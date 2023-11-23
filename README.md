# Autentisering-AWS-Setting

- Navigera till ditt projekt i terminalen.
- Börja med att installera Middy, ett middleware som hanterar det som skall hända före och efter vår lambda-funktion körs `npm install @middy/core`
- Installera också JSON webToken `npm install jsonwebtoken`
- Vi kommer också att behöva kryptera lösenord, så vi använder oss av bcrypt-biblioteket `npm install bcryptjs`
- Vi vill också kunna skapa unika användar-id:n, så vi installerar ett bibliotek för det också `npm install nanoid`

### I serverless.yml:
- Vi lägger till en ny tabell till databasen för att lagra våra användare:
Under:
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
Exempel:
```yml
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

    // check if username already exists
    // if username exists -> return { success: false, message: 'username already exists'}

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
Skicka med en JSON med data för det nya kontot:
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
Exempel:
```yml
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
Skicka med en JSON med username och password:
```yml
{
  "username": "sis",
  "password": "sita123"
}
```




