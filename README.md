Wallet container allows to store and manage user site balance: money, bonus, points, etc.

Installation
============

```bash
docker run \
-p 80:80/tcp \
-e WALLET_HOST=example.com \
-e WALLET_ACCOUNTS=money,bonus,points \
-e WALLET_COMMIT_TIMEOUT=86400 \
-e WALLET_COMMIT_WEBHOOK=http://my-site.com/check-transaction \
-v postgresql:/var/lib/postgresql \
-d perfumerlabs/wallet:1.0.0
```

Environment variables
=====================

- WALLET_HOST - server domain (without http://). Required.
- WALLET_ACCOUNTS - list of available account names separated by comma (see below). Optional. The default value is "balance".
- WALLET_COMMIT_TIMEOUT - within this term (in seconds) transaction must be committed, otherwise it will be rollbacked (see below). Optional. The default value is 86400.
- WALLET_COMMIT_WEBHOOK - URL which will be used to send requests about pending transactions (see below). Optional. If not specified, no action about pending transactions is performed.

Volumes
=======

- /var/lib/postgresql - PostgreSQL data directory.

If you want to make any additional configuration of container, mount your bash script to /opt/setup.sh. This script will be executed on container setup.

User entries
============

There is no API to create or delete users. It is done to have no need to migrate huge databases of users to Wallet. It is assumed, that every user name exists with zero balance.
 
After first transaction user entry will be created in the database automatically. Balance status request of non-existent user will not result in an error. A response with zero balance will be returned instead.  

Balance accounts
================

Money is obvious balance type, that may have a user. But in real site there can be a number of other units. For instance, it may be "bonus" balance, or you may enroll to user some kind of "points". Also money may be of different types: "dollars" and "euros".

That's why Wallet has no built-in balance types. You specify balance types by your own within container setup. By default it is just "balance". If you want to specify number of units, provide to WALLET_ACCOUNTS list of names separated by comma.

Transaction workflow
====================

Every transaction consists of 2 parts: creation and commit (or rollback) stages. Typical scheme of transaction usage in your code is following:

1. Create transaction. Block money, in other words.
2. Do your custom logic (write down purchase entries, register bonus gifts, etc.). Store somewhere transaction ID on your side.
3. Commit transaction. Complete purchase and charge money.

Between 1 and 2, 2 and 3 steps there can be failures (because HTTP requests are not reliable and can fail due to network unavailability).

###### Failure between 1 and 2 steps

In this case transaction is created in Wallet, but no purchase is executed in the application. Wallet considers this transaction as balance hold. Money is frozen, but not spent. So this transaction is not completed and Wallet will wait for commit or rollback.

If no commit or rollback action is performed within WALLET_COMMIT_TIMEOUT time, Wallet rollbacks the transaction and releases money.

###### Failure between 2 and 3 steps

In this case transaction is created in Wallet, a purchase is executed in the application, but money is not charged completely.

To fix this issue, specify WALLET_COMMIT_WEBHOOK. Wallet will send repeating requests to this URL with uncommitted transaction ID (while WALLET_COMMIT_TIMEOUT term is not expired). For example, if WALLET_COMMIT_WEBHOOK equals to http://my-site.com/check-transaction, then the request will be GET http://my-site.com/check-transaction?id=<TRANSACTION_ID>. If you want to commit the transaction return 201 status code (body doesn't matter). If you want to rollback the transaction, return 204 status code. Any other response will be considered as failure, and request will be repeated in some time. That's why you have to preserve transaction ID on your side.

API Reference
=============

###### Get all user accounts

GET /user/{user}/accounts

- {user} may have letters, digits, underscore and hyphen.
- Account balance is float variable.

Response example:

```javascript
{
    "status": true,
    "accounts": {
        "money": 100.5,
        "bonus": 400,
        "points": 50
    }
}
```

###### Create transaction

POST /transaction

- If you want to autocommit this transaction, provide "commit = true". The defailt value is "false".
- If you want to raise an error, if user has not enough balance to process transaction, provide "validate=true". The default value is "false".

Request body example:

```javascript
{
    "user": "user",
    "account": "money",
    "amount": -100,
    "validate": true,
    "commit": true
}
```

Response example:

```javascript
{
    "status": true,
    "transaction": {
        "id": 1,
        "user": "user",
        "account": "money",
        "amount": -100,
        "state": "pending",
        "created_at": "2018-12-19 14:00:00",
        "committed_at": null,
        "rollbacked_at": null
    }
}
```

###### Commit transaction

POST /transaction/{id}/commit

Response example:

```javascript
{
    "status": true,
    "transaction": {
        "id": 1,
        "user": "user",
        "account": "money",
        "amount": -100,
        "state": "committed",
        "created_at": "2018-12-19 14:00:00",
        "committed_at": "2018-12-20 14:00:00",
        "rollbacked_at": null
    }
}
```

###### Rollback transaction

POST /transaction/{id}/rollback

Response example:

```javascript
{
    "status": true,
    "transaction": {
        "id": 1,
        "user": "user",
        "account": "money",
        "amount": -100,
        "state": "rollbacked",
        "created_at": "2018-12-19 14:00:00",
        "committed_at": null,
        "rollbacked_at": "2018-12-20 14:00:00"
    }
}
```

###### List transactions

GET /transactions

Available GET-parameters:

- user (string): user name (letters, digits, underscore and hyphen are allowed). List users separated by comma, if you want to filter by several users.
- account (string): account name. List accounts separated by comma, if you want to filter by several accounts.
- state (string): status of transaction (pending, committed or rollbacked). List states separated by comma, if you want to filter by several states.
- created_at_lt (datetime): transaction creation less than this value.
- created_at_gt (datetime): transaction creation greater than this value.
- committed_at_lt (datetime): transaction commit less than this value.
- committed_at_gt (datetime): transaction commit greater than this value.
- rollbacked_at_lt (datetime): transaction rollback less than this value.
- rollbacked_at_gt (datetime): transaction rollback greater than this value.

Query example:

```
?user=user&account=money,bonus&state=pending,committed
```

Response example:

```javascript
{
    "status": true,
    "transactions": {
        [
            "id": 1,
            "user": "user",
            "account": "money",
            "amount": -200,
            "state": "committed",
            "created_at": "2018-12-19 14:00:00",
            "committed_at": null,
            "rollbacked_at": null
        ],
        [
            "id": 2,
            "user": "user",
            "account": "bonus",
            "amount": 500,
            "state": "pending",
            "created_at": "2018-12-20 14:00:00",
            "committed_at": null,
            "rollbacked_at": null
        ]
    }
}
```

###### Get transaction info

GET /transaction/{id}

Response example:

```javascript
{
    "status": true,
    "transaction": {
        "id": 1,
        "user": "user",
        "account": "money",
        "amount": -100,
        "state": "rollbacked",
        "created_at": "2018-12-20 14:00:00",
        "committed_at": null,
        "rollbacked_at": "2018-12-21 14:00:00"
    }
}
```
