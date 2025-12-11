---
layout: post
title:  "Two Phase Commit"
date:   2025-11-26 10:00:00 +0100
categories: [transaction 2pc distributed]
---

A two phase commit (2PC) is a distributed algorithm to ensure that all processes of a transaction commit together.
It is a way to enforce atomicity in a distributed system.

A 2PC execution has 2 phases: a prepare phase and a commit or rollback phase.
During the prepare phase, a coordinator will ask each process if they can process the transaction.
Each process will vote yes/no. The coordinator is responsible for deciding if the transaction should be committed or aborted.
Generally, if all processes vote yes, the transaction is committed. If any vote no, it is aborted.

To illustrate this algorithm, we will go over a basic implementation of it. This implementation does not consider edge cases,
like persistent failure of a node or failure of the coordinator.

---

Let's consider an order management system, composed of a payment service, a delivery scheduler service, a inventory service and an order management service.

* The **payment service** is responsible for handling customer payments, when an order is placed.
* The **inventory service** is responsible for managing inventory and reserving the product for the customer.
* The **delivery service** coordinates deliveries of products to customers.
* The **order management service** oversees the order and interacts with the other services as needed.

This is one example of where we can use 2PC. We want to ensure that if any service is unable to process an order we will abort the entire order.
For example, we don't want to charge a customer if there is not enough stock of a product to fulfill the order.

The order management service will act as our coordinator, responsible for requesting other services and ultimately deciding if we should commit or rollback.
Each service will have a prepare and commit method, that can be called by the coordinator.
The coordinator will decide to commit as long as all processes vote yes.

The basic flow will be:
```rust

#[derive(Serialize, Deserialize, Clone)]
struct OrderRequest {
    address: String,
    price: f32,
    product: String,
    quantity: f32
}

#[async_trait]
trait Process: Send + Sync {
    async fn prepare(&self, request: OrderRequest) -> Result<String, Error>;
    fn commit(&self, id: String);
    fn rollback(&self, id: String);
}

#[derive(Default)]
struct Coordinator {
    processes: Vec<Box<dyn Process>>,
    data: Arc<Mutex<HashMap<String, OrderRequest>>>,
}

impl Coordinator {
    async fn process_order(&self, request: OrderRequest) {
        let mut ok = true;
        let mut ids = Vec::new();
        for process in self.processes.iter() {
            let r = process.prepare(request.clone()).await;
            ok = ok && r.is_ok();
            if r.is_ok() {
                ids.push((process, r.ok().unwrap()))
            }
        }

        if (ok) {
            for (process, id) in ids {
                process.commit(id)
            }
        } else {
            for (process, id) in ids {
                process.rollback(id)
            }
        }
    }
}
```

Let's go over each service functionality before joining them in the coordinator

## Inventory Service

For the inventory service, we will be having a simplified in memory map with the current products. 
The available API will be:

* `/reserve` which reserves a specific amount of a product, so it is not available for other customers
* `/commit` will remove the product from inventory permanently 
* `/refill` which allows us to add more quantity of a product. This would be a backoffice route, for when new products are stocked

```rust
#[macro_use]
extern crate rocket;

use rocket::State;
use rocket::serde::json::Json;
use rocket::serde::{Deserialize, Serialize};
use std::collections::HashMap;
use std::sync::{Arc, Mutex};
use uuid::Uuid;

#[derive(Serialize, Deserialize)]
struct Message {
    quantity: usize,
}

#[derive(Serialize, Deserialize)]
struct Response {
    id: String,
    error: Option<String>,
}

struct DB {
    inventory: HashMap<String, usize>,
    reservations: HashMap<String, usize>,
}

type AppState = Arc<Mutex<DB>>;

#[post("/reserve/<id>", data = "<qty>")]
fn reserve(db: &State<AppState>, id: &str, qty: Json<Message>) -> Json<Response> {
    let db = &mut db.lock().unwrap();
    let reservation_id = Uuid::new_v4();

    let existing_qty = *db.inventory.get(id).or(Some(&0usize)).unwrap();
    if existing_qty <= qty.quantity {
        Json(Response {
            id: reservation_id.to_string(),
            error: Some(format!(
                "Not enough quantity of item {} (current qty: {})",
                id, existing_qty
            )),
        })
    } else {
        db.reservations.insert(String::from(id), qty.quantity);
        db.inventory.insert(reservation_id.to_string(), existing_qty - qty.quantity);
        Json(Response {
            id: reservation_id.to_string(),
            error: None,
        })
    }
}

#[post("/commit/<id>")]
fn commit(db: &State<AppState>, id: &str) {
    let reservations = &mut db.lock().unwrap().reservations;
    reservations.remove(id);
}

#[put("/refill/<id>", data = "<qty>")]
fn refill(db: &State<AppState>, id: &str, qty: Json<Message>) -> String {
    let data = &mut db.lock().unwrap().inventory;
    let inventory = data.get(id).or(Some(&0usize)).unwrap();
    data.insert(String::from(id), *inventory + qty.quantity);
    format!("Added {} items of {}", qty.quantity, id)
}

#[launch]
fn rocket() -> _ {
    rocket::build()
        .configure(rocket::Config::figment().merge(("port", 7050)))
        .manage(Arc::new(Mutex::new(DB {
            inventory: HashMap::<String, usize>::new(),
            reservations: HashMap::<String, usize>::new(),
        })))
        .mount("/", routes![reserve, commit, refill])
}
```

As for an example usage:
```
curl --location --request PUT 'http://localhost:7050/refill/bananas' \
--header 'Content-Type: application/json' \
--data '{"quantity": 100}'

Added 100 items of bananas

curl --location 'http://localhost:7050/reserve/bananas' \
--header 'Content-Type: application/json' \
--data '{
    "quantity": 10
}'

{
    "id": "10275e62-1f5a-42bc-9ecc-a0f7fc20e6f4",
    "error": null
}

curl --location --request POST 'http://localhost:7050/commit/10275e62-1f5a-42bc-9ecc-a0f7fc20e6f4'
```

## Delivery Service

For the delivery service, we will store the list of deliveries and generate an ETA on delivery.
The available API will be:

* `/schedule` which schedules a delivery and generate an ETA
* `/commit` will confirm the delivery and update the ETA if needed


```rust
use rocket::serde::json::Json;
use rocket::{State, launch, post, routes};
use serde::{Deserialize, Serialize};
use std::collections::HashMap;
use std::sync::{Arc, Mutex};
use uuid::Uuid;

#[derive(Serialize, Deserialize)]
struct DeliveryRequest {
    address: String
}

#[derive(Serialize, Deserialize, Clone)]
struct DeliveryResponse {
    id: String,
    address: String,
    date: String
}

struct DB {
    data: HashMap<String, DeliveryResponse>,
    confirmed: Vec<String>
}

type AppState = Arc<Mutex<DB>>;

#[post("/schedule", data = "<body>")]
fn delivery(db: &State<AppState>, body: Json<DeliveryRequest>) -> Json<DeliveryResponse> {
    let data = &mut db.lock().unwrap().data;
    let id = Uuid::new_v4();
    let eta = chrono::offset::Local::now()
        .checked_add_days(chrono::Days::new(5))
        .unwrap();
    let response = DeliveryResponse {
        address: body.address.clone(),
        date: eta.to_rfc2822(),
        id: id.to_string()
    };
    data.insert(
        id.to_string(),
        response.clone(),
    );
    Json(response)
}

#[post("/confirm/<id>")]
fn confirm(db: &State<AppState>, id: &str) -> Json<DeliveryResponse> {
    let db = &mut db.lock().unwrap();
    let schedule = db.data.get(id).expect("unexpected");
    let eta = chrono::offset::Local::now()
        .checked_add_days(chrono::Days::new(5))
        .unwrap();

    let response = DeliveryResponse {
        address: schedule.address.clone(),
        date: eta.to_rfc2822(),
        id: id.to_string()
    };
    db.confirmed.push(id.to_string());
    db.data.insert(
        id.to_string(),
        response.clone(),
    );
    Json(response)
}

#[launch]
fn rocket() -> _ {
    rocket::build()
        .configure(rocket::Config::figment().merge(("port", 7070)))
        .manage(Arc::new(Mutex::new(DB {
            data: HashMap::<String, DeliveryResponse>::new(),
            confirmed: Vec::new()
        })))
        .mount("/", routes![delivery, confirm])
}
```

With example usage:

```
curl --location 'http://localhost:7070/schedule' \
--header 'Content-Type: application/json' \
--data '{
    "address": "Rua da avenida, sitio do lugar"
}
'

{
    "id": "b30df39c-ece8-424d-a943-7b623cfe0047",
    "address": "Rua da avenida, sitio do lugar",
    "date": "Mon, 1 Dec 2025 17:24:16 +0000"
}

curl --location --request POST 'http://localhost:7070/confirm/b30df39c-ece8-424d-a943-7b623cfe0047'
{
    "id": "b30df39c-ece8-424d-a943-7b623cfe0047",
    "address": "Rua da avenida, sitio do lugar",
    "date": "Mon, 1 Dec 2025 17:24:22 +0000"
}
```

## Payment Service

For the payment service, we will assume that it requires a third party system, and we cannot prepare it.
So instead, we will do the transaction and reverse it in case of rollback. The commit itself will do nothing

* `/payment` which will do the transaction
* `/revese` which will undo the transaction

```rust
#[macro_use] extern crate rocket;

use std::collections::HashMap;
use std::sync::{Arc, Mutex};
use rocket::serde::{Deserialize, Serialize};
use rocket::serde::json::Json;
use rocket::State;
use uuid::Uuid;

#[derive(Serialize, Deserialize)]
struct PaymentRequest {
    account: usize,
    value: usize
}

struct DB {
    data: HashMap<String, PaymentRequest>
}

type AppState = Arc<Mutex<DB>>;

#[post("/payment", data = "<body>")]
fn payment(db: &State<AppState>, body: Json<PaymentRequest>) -> String {
    let data = &mut db.lock().unwrap().data;
    let id = Uuid::new_v4();
    // This could be a call to some 3rd party provider, to charge the value
    data.insert(id.to_string(), PaymentRequest{account: body.account, value: body.value});
    format!("Payment ID: {}, (Acc: {}, Value: {})", id.to_string(), body.account, body.value)
}

#[post("/payment/<id>/reversal")]
fn reversal(db: &State<AppState>, id: String) -> String {
    let data = &mut db.lock().unwrap().data;
    let payment = data.remove(&id);
    match payment {
        Some(p) =>
            format!(
                "Reversed payment ID: {}, (Acc: {}, Value: {})",
                id.to_string(),
                p.account,
                p.value
            ),
        None => format!("Payment {} not registered. Command ignored!", id)
    }
}

#[launch]
fn rocket() -> _ {
    rocket::build()
        .configure(rocket::Config::figment().merge(("port", 8060)))
        .manage(Arc::new(Mutex::new(DB {data: HashMap::<String, PaymentRequest>::new()})))
        .mount("/", routes![payment, reversal])
}
```

Example usage:

```
curl --location 'http://localhost:8060/payment' \
--header 'Content-Type: application/json' \
--data '{
    "account": 1,
    "value": 2000
}
'
Payment ID: eb82f3a0-0fc5-4824-a0d7-c8765a23713f, (Acc: 1, Value: 2000)

curl --location --request POST 'http://localhost:8060/payment/eb82f3a0-0fc5-4824-a0d7-c8765a23713f/reversal'
Reversed payment ID: eb82f3a0-0fc5-4824-a0d7-c8765a23713f, (Acc: 1, Value: 2000)
```

## Order Management Service

Finally, the order management service will act as our coordinator.
It will call other services and ultimately make the decision to commit or abort the transaction.

The main process code will be:

```rust
mod processes;

use rocket::serde::json::Json;
use rocket::{State, launch, post, routes, tokio::sync::Mutex};
use serde::{Deserialize, Serialize};
use std::collections::HashMap;
use std::io::Error;
use std::sync::Arc;
use uuid::Uuid;
use async_trait::async_trait;

#[derive(Serialize, Deserialize, Clone)]
struct OrderRequest {
    address: String,
    price: f32,
    product: String,
    quantity: f32
}

#[async_trait]
trait Process: Send + Sync {
    async fn prepare(&self, request: OrderRequest) -> Result<String, Error>;
    fn commit(&self, id: String);
    fn rollback(&self, id: String);
}

#[derive(Default)]
struct Coordinator {
    processes: Vec<Box<dyn Process>>,
    data: Arc<Mutex<HashMap<String, OrderRequest>>>,
}

impl Coordinator {
    async fn process_order(&self, request: OrderRequest) {
        let mut ok = true;
        let mut ids = Vec::new();
        for process in self.processes.iter() {
            let r = process.prepare(request.clone()).await;
            ok = ok && r.is_ok();
            if r.is_ok() {
                ids.push((process, r.ok().unwrap()))
            }
        }

        if (ok) {
            for (process, id) in ids {
                process.commit(id)
            }
        } else {
            for (process, id) in ids {
                process.rollback(id)
            }
        }
    }
}

#[post("/order", data = "<body>")]
async fn order(coord: &State<Coordinator>, body: Json<OrderRequest>) -> String {
    let data = &mut coord.data.lock().await;
    let id = Uuid::new_v4();
    data.insert(
        id.to_string(),
        OrderRequest {
            address: body.address.clone(),
            price: body.price,
            product: body.product.clone(),
            quantity: body.quantity,
        },
    );

    coord.process_order(OrderRequest {
        address: body.address.clone(),
        price: body.price,
        product: body.product.clone(),
        quantity: body.quantity,
    }).await;

    format!(
        "Order ID: {}, (Product: {}, Price: {}, Address: {})",
        id.to_string(),
        body.product,
        body.price,
        body.address
    )
}

#[launch]
fn rocket() -> _ {
    let coord = Coordinator {
        processes: vec![
            Box::new(processes::InventoryClient::default())
        ],
        ..Default::default()
    };
    rocket::build()
        .configure(rocket::Config::figment().merge(("port", 8090)))
        .manage(coord)
        .mount("/", routes![order])
}
```

Now, to expand this, let's create each service http client.
With the api already defined, we just need to call each service for the prepare and commit phase:

### Inventory
```rust
use std::io::{Error, ErrorKind};
use async_trait::async_trait;
use reqwest::{Client};
use rocket::serde::{Deserialize, Serialize};
use crate::{OrderRequest, Process};

#[derive(Serialize, Deserialize)]
struct IDResponse {
    id: String,
}

#[derive(Serialize, Deserialize)]
struct InventoryRequest {
    quantity: usize,
}

pub(crate) struct InventoryClient {
    base_url: String,
    client: Client
}

impl Default for InventoryClient {
    fn default() -> Self {
        InventoryClient{
            base_url: String::from("http://localhost:7050/"),
            client: Client::new()
        }
    }
}

#[async_trait]
impl Process for InventoryClient {
    async fn prepare(&self, request: OrderRequest) -> Result<String, Error> {
        let res = self.client
            .post(format!(
                "{}/reserve/{}",
                self.base_url,
                request.product
            ))
            .json(&InventoryRequest { quantity: request.quantity })
            .send()
            .await;

        if res.is_err() {
            return Err(Error::new(ErrorKind::Other, res.err().unwrap().to_string()));
        }
        let r = res.ok().unwrap().json::<IDResponse>().await.unwrap();
        Ok(r.id)
    }

    async fn commit(&self, id: String) {
        self.client
            .post(format!(
                "{}/commit/{}",
                self.base_url,
                id
            ))
            .send()
            .await.expect("Unexpected commit error");
    }

    async fn rollback(&self, id: String) {
        todo!()
    }
}
```

### Delivery
```rust
pub(crate) struct DeliveryClient {
    base_url: String,
    client: Client
}

impl Default for DeliveryClient {
    fn default() -> Self {
        DeliveryClient{
            base_url: String::from("http://localhost:7070/"),
            client: Client::new()
        }
    }
}

#[derive(Serialize, Deserialize)]
struct DeliveryRequest {
    address: String
}

#[async_trait]
impl Process for DeliveryClient {
    async fn prepare(&self, request: OrderRequest) -> Result<String, Error> {
        let res = self.client
            .post(format!(
                "{}/schedule",
                self.base_url,
            ))
            .json(&DeliveryRequest { address: request.address.clone() })
            .send()
            .await;

        if res.is_err() {
            return Err(Error::new(ErrorKind::Other, res.err().unwrap().to_string()));
        }
        let r = res.ok().unwrap().json::<IDResponse>().await.unwrap();
        Ok(r.id)
    }

    async fn commit(&self, id: String) {
        self.client
            .post(format!(
                "{}/confirm/{}",
                self.base_url,
                id
            ))
            .send()
            .await.expect("Unexpected commit error");
    }

    async fn rollback(&self, id: String) {
        todo!()
    }
}
```

### Payment
```rust
pub(crate) struct PaymentClient {
    base_url: String,
    client: Client
}

impl Default for PaymentClient {
    fn default() -> Self {
        PaymentClient{
            base_url: String::from("http://localhost:8060/"),
            client: Client::new()
        }
    }
}

#[derive(Serialize, Deserialize)]
struct PaymentRequest {
    account: usize,
    value: usize
}

#[async_trait]
impl Process for PaymentClient {
    async fn prepare(&self, request: OrderRequest) -> Result<String, Error> {
        let res = self.client
            .post(format!(
                "{}/payment",
                self.base_url,
            ))
            .json(&PaymentRequest { account: 10, value: request.price as usize })
            .send()
            .await;

        if res.is_err() {
            return Err(Error::new(ErrorKind::Other, res.err().unwrap().to_string()));
        }
        let r = res.ok().unwrap().json::<IDResponse>().await.unwrap();
        Ok(r.id)
    }

    async fn commit(&self, _: String) {
        // Do nothing, because service cannot reserve
    }

    async fn rollback(&self, id: String) {
        todo!()
    }
}
```

When everything goes well and all services are successful, all 3 prepare and commit methods are called.

# Rollback Scenario

If one of the services votes NO or has an error, then the remaining services are not called.
The ones already called will have to go through a rollback phase, to restore the system to the pre-order state.

Let's review each service and understand how we can rollback an action made on the prepare stage.

## Delivery

For the delivery service, we have an internal data struct and a confirmed data struct.
To rollback, we can simply remove the entry from the internal data struct.

```rust
#[post("/rollback/<id>")]
fn rollback(db: &State<AppState>, id: &str) {
    let db = &mut db.lock().unwrap();
    db.data.remove(id);
}
```

And update the order management method to call this new route:

```rust
#[async_trait]
impl Process for DeliveryClient {
    //...
    async fn rollback(&self, id: String) {
        self.client
            .post(format!(
                "{}/rollback/{}",
                self.base_url,
                id
            ))
            .send()
            .await.expect("Unexpected rollback error");
    }
}
```

## Inventory

For the inventory service, we also store a reservations and an inventory data struct.
When we prepare, we are adding a reservation entry and reducing the inventory size.
Committing just removes the reservation.
So, to rollback we need to remove the reservation and restore the inventory.

```rust
#[post("/rollback/<id>")]
fn rollback(db: &State<AppState>, id: &str) {
    let mut data = db.lock().unwrap();
    let (reservation, qty) = data.reservations.get(id).expect("invalid rollback id").clone();
    let existing_qty = data.inventory.get(&reservation).expect("invalid rollback id").clone();
    data.inventory.insert(reservation, existing_qty + qty);
}
```

And adapt the management method to call this new rollback method

```rust
#[async_trait]
impl Process for DeliveryClient {
    // ...
    async fn rollback(&self, id: String) {
        self.client
            .post(format!(
                "{}/rollback/{}",
                self.base_url,
                id
            ))
            .send()
            .await.expect("Unexpected rollback error");
    }
}
```

## Payment
Finally, for payments, we don't have any temporary data structure.
There might be such cases, where we will have to undo an action while to commit doesn't do much.

For our case, let's create such a method

```rust
#[post("/payment/<id>/reversal")]
fn reversal(db: &State<AppState>, id: String) -> String {
    let data = &mut db.lock().unwrap().data;
    let payment = data.remove(&id);
    match payment {
        Some(p) =>
            format!(
                "Reversed payment ID: {}, (Acc: {}, Value: {})",
                id.to_string(),
                p.account,
                p.value
            ),
        None => format!("Payment {} not registered. Command ignored!", id)
    }
}
```

and update our management service

```rust
#[async_trait]
impl Process for PaymentClient {
    // ...
    async fn rollback(&self, id: String) {
        self.client
            .post(format!(
                "{}/payment/{}/reversal",
                self.base_url,
                id
            ))
            .send()
            .await.expect("Unexpected rollback error");
    }
}
```

In real world scenarios, this type of limitation could be even more problematic.
We might have a third party service to interact with, and such reversals will still show up on 
payment extracts (as a debit and a refund, for example).

Also, we have ignored other limitations of this: what happens if the management service goes down?
Can we have transactions that are left in the prepare stage for a long time? For example, with inventory this
could make us have less inventory than what is actually available. There are ways to mitigate this, of course,
but it is important to keep in mind that these edge cases can come up when applying this pattern.

_The full source code for this article is in [my github repo](https://github.com/SLourenco/blog-demos)._
