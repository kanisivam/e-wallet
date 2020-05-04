# e-wallet
Python Django Mobile Wallet Application

MobileWallet2020 Application Uses:
--------------------------------------------------
Python v3.6.0
Django v2.11.0
Django REST Framework v3.11
MySql(Maria DB) / SQLLite


1. Created necessary table's
Table and Column:
i. user - id, mobileno(primary key), email_id, name, user_type, password(encrypted and hashed), is_active, created_date, last_login
ii. wallet - id, user_hash, currency, balance, last_updated_time
iii. wallethistory(audit table for wallet) - id, user_hash, currency, old_balance, new_balance, last_updated_time
iv. wallettransaction(transaction level details) - id, user_hash, transaction_type, transaction_status, recipient, reason, currency, amount, transaction_date
Id fields are auto increment.

Password - hashed.
Currency - with an idea for wallet to support multiple currency.
1. Implemented : Multi-wallet to support multi-currency. Fund transfer from SGD to SGD or USD to USD.
2. We can extend to support conversion as well with this in future.
user_hash in wallet table is hashed value of mobieno+currencytype from user table with an idea to keep the wallet balance to an user masked.


2. /auth/getToken [POST]- endpoint called to login:

Request - {"username": "+65XXXXXXX", "password": "xxxxxxxxxxxx"}
Response - {"token": "17321cc6f4e7af4c1e9136ff14e4f86c41796db2"}
is_active column in user table to be true(1).
The generated token is sent in header as "Authorization"(key) - "GeneratedToken"(Value).


3. The token is validated before processing any service in transfer service.
If not valid "Invalid token" error message is sent as response. If valid user can access the endpoints.

4. Authorization & Permission validation:
Assuming three types of user(User, admin and super user) can exist in system for now, the endpoints an user can access is configured in config file


PERMISSION = {'USER':
                  {
                      'POST':['/transfer/getBalance' , '/transfer/getTransaction','/transfer/transferFund'],
                      'GET': []
                  },
              'ADMIN':
                  {
                      'POST': ['/transfer/getBalance' , '/transfer/getTransaction' , '/transfer/transferFund', '/transfer/getResourceUsage'],
                      'GET': []
                  },
              'SUPER_USER':
                    {
                        'POST': ['/transfer/getBalance' , '/transfer/getTransaction', '/transfer/transferFund', '/transfer/getResourceUsage', 'wallet/adduser', 'wallet/changeuser'],
                        'GET': []
                    }

}
(Can be configured in db too. Since we are concentrating mostly on transfer service, made this minimal effort.)

5. Get Balance
Endpoint - /transfer/getBalance [POST]
Request - Token is decrypted and mobile number is fetched from it to get the balance.  {"currency": "SGD"}
Response - [{"currentBalance": 170, "currency": "SGD"}]
If token is invalid an error message is sent.
If not, mobile number from token is hashed to form user_hash and balance is queried from wallet table.
Unit testing done.

6. Get Transaction:
Endpoint - /transfer/getTransaction [POST]
Request - {"mobileno": "+65XXXXXXX","startdate" : "22-04-2020","enddate" :"27-04-2020", "currency": "SGD" }
Response - [{"sender": "+65XXXXXXX","transactionDate": "2020-04-27T06:29:20.141506Z","transactionType": "CREDIT","amount": 10,"currency": "SGD","recipient": "+65XXXXXXX","transactionStatus": "SUCCESS","reason": "EMI"},....]
Token is decrypted and mobile number in request is validated if user is trying to get transaction of his/her account.
Unit testing done.

7. Transfer Fund:
Endpoint - /transfer/transferFund [POST]
Request - {"recipient": "+91XXXXXX", "transactionType": "CREDIT" , "amount": "10" ,"currency":"SGD" , "reason":"EMI" }
Response - {"transactionStatus": "SUCCESS"}
1. Sender fetched from token.
2. Query wallet table with the hashed value of mobile number and currency. If no row returned error message is returned.
3. Check transfer amount with the available balance. If balance is not sufficient to make transfer, "Insufficient balance" error message is returned.
4. Check if recipient is valid and active. If not error message is returned.
5. Entry is made to transaction table with in progress status for the sender.
6. Within transaction scope
i. Subtract transfer amount from the sender's balance in wallet table.
ii. Add entry to wallet history table for sender row update.
iii. Add transfer amount to recipient in wallet table.
iv. Add entry to wallet history table for receiver row update.
v. Add entry for recipient in transaction table with status success.
7. Update transaction status as success for sender entry.
8. Mail will be sent to sender and receiver with updated balance and received amount respectively.
(Transaction & updated wallet balance status will be notified to sender&recipient through email.)


8. API Usage Tracking. 
DRF-tracking in Django.
Endpoint - /transfer/getResourceUsage [POST]
it will list out the api resource usage (response time, params, status code etc)
it will help management to get statistical report
