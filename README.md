## Nerdery DB Challange \#3 \- ERD.
![Proposed Diagram](/ERD-Diagram.png)

Available in dbdiagram.io [https://dbdiagram.io/d/nerdery_challenge_3-674890aae9daa85aca0aa678](https://dbdiagram.io/d/nerdery_challenge_3-674890aae9daa85aca0aa678)

REST swagger documentation available here: [https://ezrillex.github.io/DB-Nerdery-Challenge-ERD/](https://ezrillex.github.io/DB-Nerdery-Challenge-ERD/)

![Proposed GraphQL](/graphql_visualized.png)
[Open in new tab](https://raw.githubusercontent.com/ezrillex/DB-Nerdery-Challenge-ERD/refs/heads/main/graphql_visualized.png)
# GraphQL --------------------------------
## Mandatory Features

1. Authentication endpoints (sign up, sign in, sign out, forgot, reset password)
- Sign Up is **createUser** mutation
- Sign In is **loginUser** mutation
- Sign Out is **logoutUser** mutation
- Forgot is **forgotPassword** mutation
- Reset Password is **resetPassword** mutation
2. List products with pagination
- Provided by **searchProducts** query. This accepts pagination parameters:
  - first: Int!  
  - offset: Int!  
  - pageSize: Int
3. Search products by category
- Provided by **searchProducts** query. This accepts search parameters: -
  - categoryFilter: [ID!]
  - search: String 
  - likedOnly: Boolean
  - sort: String
4. Add 2 kinds of users (Manager, Client)
- This is managed by the **Users** type that has **Roles** type relation. 
5. As a Manager I can:
    * Create products
    * Update products
    * Delete products
    * Disable products
    * Show clients orders
    * Upload images per product.
6. As a Client I can:
    * See products
    * See the product details
    * Buy products
    * Add products to cart
    * Like products
    * Show my order

BOTH 5 and 6 are covered by **userHasPermission** query which takes the user id and the permission id we want to acess and returns a boolean that it has or doesnt have such permission. If no id is provided anonymous/public permissions are applied. 

7. The product information(included the images) should be visible for logged and not logged users
- The endpoints **getFile**, **getFiles**, and **searchProducts** are accesible to anonymous users. 
8. Stripe Integration for payment (including webhooks management)

    The payment lifecycle has many steps this are fullfilled by endpoints:
   - (1) A payment is created in our system and sent for creation in stripe with **createPayment** mutation. 
   - (2a) Once the user fills in card info this is sent to stripe in **updatePaymentDetails** mutation.
   - (2b) If the user confirms in the final step to place order a **updatePaymentDetails** mutation is called with the confirm flag to true, this will signal stripe to go ahead and charge it.
   - (3) We wait for stripe to send info about how it went in the **stripePaymentWebhook** mutation which is behind a REST front as stipe doesn't support GQL.
   - (4a) Workers actively scan for new payments to process in **getPaymentsToProcess** query.
   - (4b) With their list they go one by one using **lockPayment** mutation to get the data and to lock the payment for being processed by some other worker. 
   - (4c) Once the worker has run some business logic like set order to paid, or issue a refund. It uses **processPayment** mutation to update the status accordingly and close the payment cycle. 

## Extra points

* When the stock of a product reaches 3, notify the last user that liked it and not purchased the product yet with an email.
  - A cron job hits the **lowStockPromotionsCronJob** endpoint mutation that triggers such logic. , A job is added to the email services by using the **createEmailJob** mutation.
* Send an email when the user changes the password
  - When the **resetPassword** mutation (which could be used for  changing the password) is used, a job is added to the email services by using the **createEmailJob** mutation.


# REST -----------------------------------
1. Authentication endpoints (sign up, sign in, sign out, forgot, reset password)
- Sign Up is **/users POST**
- Sign In is **/auth/login POST**
- Sign Out is **/auth/logout POST**
- Forgot is **/auth/forgot POST** 
- Reset Password is **/auth/reset POST**
2. List products with pagination
- Provided by **/products GET**. This accepts pagination parameters:
    - page: Int
    - page_size: Int!
3. Search products by category
- Provided by **/products GET**. This accepts search parameters: -
    - category : id array
    - search: String
    - likes: Boolean
    - sort_by: String
4. Add 2 kinds of users (Manager, Client)
- This is managed by the **Users** table that has **Roles, Permissions, Permits** table relations.
5. As a Manager I can:
    * Create products
    * Update products
    * Delete products
    * Disable products
    * Show clients orders
    * Upload images per product.
6. As a Client I can:
    * See products
    * See the product details
    * Buy products
    * Add products to cart
    * Like products
    * Show my order

**BOTH** **5** and **6** are covered by each endpoint which checks permits table to see if role has such a permission and returns 401 or 403 depending on the scenario. 

7. The product information(included the images) should be visible for logged and not logged users
- The endpoints **/products GET** and **/products/id GET** includes the image cdn links by default, this endpoint is accessible to anonymous users. Such cdn links are public. No private links should exist in this table. Such requirement would be preferrable to build a different table like private_files.
8. Stripe Integration for payment (including webhooks management)

   The payment lifecycle has many steps this are fullfilled by endpoints:
    - (1) A payment is created in our system and sent for creation in stripe with **/payments POST** endpoint. 
    - (2a) Once the user fills in card info this is sent to stripe in **/payments/id PUT** endpoint.
    - (2b) If the user confirms in the final step to place order a **/payments/id PUT** endpoint is called with the confirm flag to true, this will signal stripe to go ahead and charge it.
    - (3) We wait for stripe to send info about how it went in the **/webhook POST endpoint**.
    - (4a) Workers actively scan for new payments to process in **/payments GET** worker endpoint, which has pre-configured filters. 
    - (4b) With their list they go one by one using **/webhook/id POST** to get the data and to lock the payment for being processed by some other worker.
    - (4c) Once the worker has run some business logic like set order to paid, or issue a refund. It uses **/payments/id POST** to update the payment and **/webhook/id POST** to update the status accordingly and close the payment cycle.
    - Additional calls to other endpoints might be done depending on business logic. Such as setting an order status to paid or sending update emails.

## Extra points

* When the stock of a product reaches 3, notify the last user that liked it and not purchased the product yet with an email.
    - A cron job hits the **/promotions POST** endpoint mutation that triggers such logic. , A job is added to the email services by using the **createEmailJob** mutation.
* Send an email when the user changes the password
    - When the **/auth/reset** POST endpoint (which could be used for  changing the password) is used, a job is added to the email services by using the **/emails POST** endpoint.
# DB Design (outdated) -----------------------
## **Mandatory Features**

### 1\. Authentication endpoints (sign up, sign in, sign out, forgot, reset password)  
The users follow the user lifecycle which covers the mentioned operations.   
![User Lifecyle](/image1.png)  
Here is how each step of the lifecycle interacts with the proposed database.

1. Sign-up. The user fills out a form in a registration screen. When submitted the application creates a row in the **users** table. The app must hash+salt the password that is sent. The app must create a queue item on the **email\_queue** with a welcome email to the user. (Some actions could be done by a trigger but to focus on the DB lets assume the app handles such actions). If the user is of type ‘Manager’ this has to be set by the system administrator by updating the type of user to ‘manager’.   
2. Sign-in. The user fills its credentials on the login screen. When submitted the app will cryptographically compare the credentials with the password and email on the **users** table. If there is a match, but the password is wrong the app must increase **failed\_login\_attempts**. If the credentials are good, the app must update the **last\_login\_at**, set the **failed\_login\_attempts** to 0, set the **password\_reset\_attempts** to 0, and clear the **password\_reset\_attempts\_tiemstamps**.   
3. Sign-out. If the user chooses to close his user in the app. The app must update the timestamp of **last\_signout\_at.** To determine if the user needs to re-login past some time, the app can query the interval between **last\_signout\_at** and **last\_login\_at** to determine if 72 hours have passed or if the sign out is greater than sign in then the user performed a sign out operation.   
4. Forgot credentials. If the user needs to request a reset password email, the forgotten credentials flow is performed in the app. The app must increase **password\_reset\_requests** and push a timestamp into **password\_reset\_requests\_tiemstamps**. If the value is over 3 the app must check if the timestamps in the array are from the past 72 hours for example. Then it will determine if to disallow the user to request more password reset emails. If these conditions are not met, then the forgotten credentials flow can continue. The app must create a reset password email and add it to the **email\_queue**. The email must contain a unique link with a token which must also be recorded on **password\_reset\_token**.   
5. Reset Password. The user is presented with a screen to input a new password, this screen must check the token from the link is valid by looking it up on **password\_reset\_token**. Once completed the app must  hash and salt the new password and update the **password\_hash**, set null to **password\_reset\_token**, set to 0  **password\_reset\_requests** and clear the **password\_reset\_requests\_tiemstamps**. Afterwards the sign in flow is presented to the user and actions described there are also performed. 

### 2\. List products with pagination  
The app must query the **products** table with plenty of implementation options mentioned here. [https://www.citusdata.com/blog/2016/03/30/five-ways-to-paginate/](https://www.citusdata.com/blog/2016/03/30/five-ways-to-paginate/) 

From a purely db side, the **products** table must have some columns with indexes for faster queries. We need to also have **files** indexed to prevent images from taking too long to load. 

### 3\. Search products by category  
The app must query the **products** table, ordered by **categories** table. From a purely db side we need to have indexes on **categories** and **files** for faster queries. 

### 4\. Add 2 kinds of users (Manager, Client)  
This is defined by the **type** field on the **users** table. This is of type **user\_types** which is an enum that has 3 values: **manager**, **client**, and anonymous. The last one being the public user, meaning some user that has not logged in. 

For the following 5 and 6 requirements we need to define some permissions. The app must query the **permits** table and get the user row from the **users** table. With the **user\_type** the app must filter the permissions table on the **role** column. This will result in a pre defined list of permits listed in the **permissions** enum, the permissions table is configured with rows that define the following:

| Col name | id | role | permit | has\_permit | for\_table |
| :---- | :---- | :---- | :---- | :---- | :---- |
| Data type | Primary key | user\_role | permissions | boolean | varchar |
| description |  | Role of user | Kind of permit | Does it have it (yes/no) | This applies  to what table? |

Once the app has a row describing the permit it wants to check it decides based on the **has\_permission** if it can proceed or not. For each permit described next, I will specify the row that I expect to see in the **permits** table. Relevant actions have been described, other details can be checked in the DMLB notes. Fields like last\_updated\_at we assume the app will update relevant fields in their respective tables. 

The permission tables have been updated however the main change is that we no longer use enums. The rows on permits will be mostly the same. 

### 5\. As a Manager I can:  
* Create products

| role | permit | has\_permit | for\_table |
| :---- | :---- | :---- | :---- |
| manager | create | true | products |

* Update products

| role | permit | has\_permit | for\_table |
| :---- | :---- | :---- | :---- |
| manager | update | true | products |

* Delete products

| role | permit | has\_permit | for\_table |
| :---- | :---- | :---- | :---- |
| manager | delete | true | products |

* Disable products

| role | permit | has\_permit | for\_table |
| :---- | :---- | :---- | :---- |
| manager | disable | true | products |

* Show clients orders

| role | permit | has\_permit | for\_table |
| :---- | :---- | :---- | :---- |
| manager | view\_all | true | products |

* Upload images per product.

| role | permit | has\_permit | for\_table |
| :---- | :---- | :---- | :---- |
| manager | upload | true | products |

### 6\. As a Client I can:  
* See products   AND  See the product details  

The app doesn’t check for permits and queries for the **products joined with** **categories** **joined with files** table. 

* Buy products  

The app performs several queries.

1. Create a new order in the **orders** table.   
2. Delete selected products from the **cart** table and create them in the **order\_details** table. One line per product with the associated **order\_id**.   
3. A payment request was sent to Stripe. Right now the order has a null **payment\_id**, this field is meant to link payment status updates to the order, the status is pending until we get any info from stripe. Regarding the request sent, this is logged in the payment\_requests table and the id is set in **payment\_request\_id** .   
4. The app adds an email to **email\_queue** with order information.   
5. The app updates the product **stock.**

* Add products to cart  

The app will create or update the **cart** table with a row where **product\_id** matches. And adjust the quantity accordingly. The cart table has one row per product. Stock validations will be performed by the app at checkout thus no need to check stock in the cart.  
* Like products  

The app will create an entry in the likes table.  
* Show my order

| role | permit | has\_permit | for\_table |
| :---- | :---- | :---- | :---- |
| client | view\_self | true | orders |

The app will check if the user has permission to see the order by checking the permits. Orders are permit checked. 

### 7\. The product information(included the images) should be visible for logged and not logged users.   
As mentioned in the client products section, the app doesn’t check for permits and queries for the **products joined with** **categories** **joined with the files** table. 

### 8\. Stripe Integration for payment (including webhooks management) 

Based on the information on [https://docs.stripe.com/payments/paymentintents/lifecycle](https://docs.stripe.com/payments/paymentintents/lifecycle) we will now use a row for each payment intent on the payments table in order to update the status of payments. This table has a row for each payment attempt. Even if stripe supports re-starting the flow if the flow is re-started from the store website then we will create a new row. Or if an attempt times out then a new one will be started. There is many scenarios detailed in the documentation. 

The app will perform payment requests and log them in the **payments** table. The id of the request log will be pushed to  **payment\_id** as in some edge scenarios a payment for an order can be retried so we could end up with multiple requests. 

When stripe hits the payment updates webhook, some fields will be unrolled for easier querying. The request will be added in **payments** which will be processed by a worker that will update the order accordingly.  For example, set **payment\_status** to paid. Or if the payment was somehow cancelled have the app add an email to **email\_queue** to notify management of the issue. The worker will also update the queue with a status of in progress and once some logic is performed with a success or fail. 

## Extra points

* When the stock of a product reaches 3, notify the last user that liked it and not purchased the product yet with an email. 

The app will check when a purchase is made or some other triggering logic. From a purely db standpoint we need to check the product **stock** field, and then filter the **likes** table with that low stock **product**, we will subsequently add to the **email\_queue** a marketing email to each client. This is done because we have the ids of the clients in the likes table and we can join to the **users** table to get their **emails**. Additionally if the email needs any images fetch them from the **files** table and set them in the **attachments** field. This holds id’s of files to be sent.

* Send an email when the user changes the password

After the steps detailed in section 1.e (reset password). We need the app to add an email to **email\_queue** notifying the user that his password was changed.

So to talk about the **email\_queue**, this is the same as the payments webhook, a **worker**(usually called mailer in this context) will process the rows in the email\_queue and update the status to successful or failed depending if the email was sent or not.
