# Autentisering-AWS-Setting

- Navigera till ditt projekt i terminalen.
- Börja med att installera Middy, ett middleware som hanterar det som skall hända före och efter vår lambda-funktion körs `npm install @middy/core`
- Installera också JSON webToken `npm install jsonwebtoken`
- Vi kommer också att behöva kryptera lösenord, så vi använder oss av bcrypt-biblioteket `npm install bcryptjs`
- Vi vill också kunna skapa unika användar-id:n, så vi installerar ett bibliotek för det också `npm install nanoid`

### I serverless.yml:
- Vi lägger till en ny tabell till databasen för att lagra våra användare:
- Under:
```yml
resources:
  Resources:
```
Lägg til t.ex.:
```yml
resources:
  Resources:
    dogsDb:
      Type: AWS::DynamoDB::Table
      Properties:
        TableName: dogs-db
        AttributeDefinitions: 
          - AttributeName: id
            AttributeType: S
        KeySchema:
          - AttributeName: id
            KeyType: HASH
        BillingMode: PAY_PER_REQUEST

    usersDb:
      Type: AWS::DynamoDB::Table
      Properties:
        TableName: accounts
        AttributeDefinitions: 
          - AttributeName: username
            AttributeType: S
        KeySchema:
          - AttributeName: username
            KeyType: HASH
        BillingMode: PAY_PER_REQUEST
```

# Skapa en signUp-funktion:
- Ny folder i funktions-foldern: `signUp`
- Nytt dokument: `index.js`
- Exempel:
```javascript
const { nanoid } = require("nanoid");
const { sendResponse } = require("../../responses");
const bcrypt = require('bcryptjs');
const AWS = require('aws-sdk');
const db = new AWS.DynamoDB.DocumentClient();


async function createAccount(username, hashedPassword, userId, firstname, lastname) {

    try {
        await db.put({
            TableName: 'accounts',
            Item: {
                username: username,
                password: hashedPassword,
                firstname: firstname,
                lastname: lastname,
                userId: userId,
            }
        }).promise();

        return {success : true, userId: userId};
    } catch (error) {
        console.log(error);
        return {success: false, message: 'Could not create account'};
    }
}

async function signup(username, password, firstname, lastname) {

    //check if username already exists
    //if username exists -> return { success: false, message: 'username already exists'}

    const hashedPassword = await bcrypt.hash(password, 10);

    const userId = nanoid();

    const result = await createAccount(username, hashedPassword, userId, firstname, lastname);
    return result;
}

exports.handler = async (event) => {
    const { username, password, firstname, lastname} = JSON.parse(event.body);

    const result = await signup(username, password, firstname, lastname);

    if (result.success)
        return sendResponse(200, result);
    else 
        return sendResponse(400, result);
}
```
### Lägg till en ny endpoint i serverless.yml:
```yml
signUp:
     handler: functions/signUp/index.handler
     events: 
      - httpApi:
          path: '/auth/signup'
          method: POST
```
- Skicka upp koden till aws `sls deploy`
- Testa med Insomnia:
- Använd den nya endpoint vi skapat `https://{din adress till aws kolla api gateway}/auth/signup`
- Skicka med en JSON med data för det nya kontot:
```yml
{
 "username": "sis",
 "password": "sita123",
 "firstname": "sita",
 "lastname": "teo"
}
```

# Skapa en login-funktion:
- Ny folder i funktions-foldern: `login`
- Nytt dokument: `index.js`
- Exempel:
```javascript
const AWS = require('aws-sdk');
const db = new AWS.DynamoDB.DocumentClient();
const bcrypt = require('bcryptjs');
const jwt = require('jsonwebtoken');
const { sendResponse } = require('../../responses');


async function getUser(username) {

    try {
        const user = await db.get({
            TableName: "accounts",
            Key: {
                username: username
            }
        }).promise();

        if (user?.Item)
            return user.Item
        else 
            return false

    } catch (error) {
        console.log(error)
        return false
    }
}

async function login(username, password) {
    const user = await getUser(username);

    if (!user) return {success: false, message: 'Incorrect username or password' };

    const correctPassword = await bcrypt.compare(password, user.password);

    if(!correctPassword) return {success: false, message: 'Incorrect username or password' };

    const token = jwt.sign({ id: user.userId, username: user.username}, "aabbcc", { expiresIn: 3600 } ); //en timme loga in

    return {success: true, token: token }

}

exports.handler = async (event) => {
    const { username, password } = JSON.parse(event.body);

    const result = await login(username, password);

    if (result.success)
        return sendResponse(200, result);
    else 
        return sendResponse(400, result);
}
```
### Lägg till en ny endpoint i serverless.yml:
```yml
 login:
    handler: functions/login/index.handler
    events:
      - httpApi:
          path: '/auth/login'
          method: POST
```
- Skicka upp koden till aws `sls deploy`
- Testa med Insomnia:
- Använd den nya endpoint vi skapat `https://{din adress till aws kolla api gateway}/auth/signup`
- Skicka med en JSON med username och password:
```yml
{
  "username": "sis",
  "password": "sita123"
}
```

# Autentisering till aws, lägga till middy som middleware
Börja med att se till att rätt version av middy är installerat i projektet. Vi har ju använt oss av commonJS tidigare och senaste versionen av middy har inte stöd för det.

### I package.json
under dependencies, se till att ändra middy-propertyn:
```yml
{...
"@middy/core": "^4.6.0"
...}
```
### I terminalen:
Kör `npm install`

### I functions/getDogs/index.js:
```javascript
const AWS = require('aws-sdk');
const { sendResponse } = require('../../responses');
const middy = require('@middy/core'); // Vi importerar middy
const { validateToken } = require('../middleware/auth');
const db = new AWS.DynamoDB.DocumentClient();



const getDogs = async (event, context) => {

  if (event?.error && event?.error === '401')
     return sendResponse(401, {success: false, message: 'Invalid token'});
    
  const {Items} = await db.scan({
    TableName: 'dogs-db'
  }).promise();

    return sendResponse(200, {success: true, dogs: Items});
}

const handler = middy(getDogs)
     .use(validateToken)
  

module.exports = { handler };
```
- Skapa en ny fil: `/functions/middleware/auth.js`

### I /functions/middleware/auth.js
- Skapa funktion för att det ska krävs en giltig inloggning för att köra getDogs.

```javascript
const jwt = require('jsonwebtoken');

const validateToken = {
    before: async (request) => {
        try {
            const token = request.event.headers.authorization.replace('Bearer ', '');

            if (!token) throw new Error();

            const data = jwt.verify(token, 'aabbcc');

            request.event.id = data.id;
            request.event.username = data.username;

            return request.response;
        } catch (error) {
            request.event.error = '401'
            return request.response;
        } 
    },
    onError: async (request) => {
        request.event.error = '401'
        return request.response;
    }
}

module.exports = { validateToken };
```
- Skicka upp koden till aws `sls deploy`

### I Insomnia:
- Börja med att logga in en existerande användare: `(Finns det inte får du skapa en!)`
- POST https://{ditt_api:s_adress}/auth/login
- Skicka med username och password:
```yml
{
  "username": "sis",
  "password": "sita123"
}
```
Du får förhoppningsvis tillbaks en respons som ser ut så här ungefär:
```yml
{
 "success": true,
 "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpZCI6IkFrTm5NeWxnRExHM19sOVROVGdFQyIsInVzZXJuYW1lIjoic2lzIiwiaWF0IjoxNzAwNTc2OTQ2LCJleHAiOjE3MDA1ODA1NDZ9.X6qYjIZ2nB3SUysO8ctgQg97jb7nlh_54sDGX-Z_6vE"
}
```
- Kopiera token
- Nu gör du ett anrop till: `GET https://{ditt_api:s_adress}/dogs`

### MEN GLÖM INTE:
- Du måste skicka med din token!
- I Insomnia, öppna fliken `auth`
- Välj `Bearer` token i drop-down-menyn.
- Klistra in din `token`.
- Tryck på send.
`Om din token är valid, så körs lambda-funktionen och listan med hundar returneras.`





