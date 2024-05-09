

1. Install mongodb

https://www.mongodb.com/docs/manual/tutorial/install-mongodb-on-ubuntu/


Note: Only install mongosh:

sudo apt-get install -y mongodb-org-shell


2.

ps aux | grep mongo
mongodb  1368919  2.5 14.1 2639876 137824 ?      Ssl  19:14   0:02 /usr/bin/mongod --config /etc/mongod.conf

/etc/mongod.conf
/var/log/mongodb
/var/lib/mongodb




use admin
db.createUser(
  {
    user: "root",
    pwd: "admin123",
    roles: [
      { role: "root", db: "admin" }
    ]
  }
)



db.getUsers({
    showCredentials: true
})



Add 

security:
    authorization: enabled

to mongo.conf


Authenticate:

mongosh --port 27017  --authenticationDatabase \
    "admin" -u "myUserAdmin" -p

or in mongosh

use admin
db.auth("myUserAdmin", passwordPrompt()) // or cleartext password




https://www.prisma.io/dataguide/mongodb

