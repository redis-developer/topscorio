Topscorio
=========

Keeps scores for gamers. 

Installing it locally
====

Make a `.env` file like so:

```
NODE_ENV=development
SERVER_PORT=5020
SESSION_REDIS=redis://default:<pass>@<url>
USERS_REDIS=redis://default:<pass>@<url>
GAMES_REDIS=redis://default:<pass>@<url>
GAME_LOGS_REDIS=redis://default:<pass>@<url>
SENDGRID_API_KEY=<SENDGRID_API_KEY>
```

Filling in database addresses, passwords and the sendgrid api key as needed. I have used a different database for each,
but I don't see a reason why it couldn't all be the same database url's if you're just testing it out.

Make sure that RediJSON is enabled for the databases.

Then in one terminal run:

```
npm install
npm run backend
```

and then in another terminal run

```
npm run frontend
```

And navigate to http://localhost:5010 in your browser.

Game example
=====

The app has an example Chess game implemented, open up multiple browser sessions to test this.

![Screenshot showing online multiplayer chess](https://i.imgur.com/59ZEyt8.png)

Architecture
=======

The frontend connects to the backend through a websocket.

The frontend sends requests to the backend, requesting subscriptions on channels, and executing commands. The backend
broadcasts channels to all connected clients, and sends messages with updates as responses to client commands.

User sessions are stored in the SESSION_REDIS database, with an expiry set.

User information is stored in USERS_REDIS, it is recommended persistence is enabled for this database.

Games are saved in GAMES_REDIS, this should also be persisted.

Game logs are instances of players using a game together. This is where all game moves and states are saved. Persistence
is also recommended here.


How data is stored
=======

## Users and sessions

Users and sessions storage is managed in `backend/authentication_store.ts`.

New sessions are stored with `SETEX`, generating a unique session token for each session. E.g.: `SETEX session-123abc 259200 {"some": "json"}`

Sessions are resumed by fetching the setting from `localStorage` and then retrieving the session from Redis with e.g.: `GET session-123abc`

New authentication tokens are stored the same way, but with a shorter expiry.

Users are stored using `JSON.SET`, using the e-mail address as a key.

Users are retrieved using `JSON.GET` using '.' as the path to retrieve the whole object.

# Games

Games and game logs storage is managed in `backend/games_store.ts`.

A cache of all games is stored in the games database at key `all-games`. Upon server start up it is first retrieved using `JSON.GET all-games .`. If it is null then it is populated with a `JSON.SET all-games . {"newest": []}` command.

Upon server startup the server subscribes to the `newest` channel on the games database, and the `open` channel on the game logs database. It does this with the `SUBSCRIBE` command like so: `SUBSCRIBE newest`.

Whenever a client shows interest in a channel, the server initiates a subscription. For example when a user joins a game with id `abc123`, the server will subscribe to that game using `SUBSCRIBE abc123`.

At the moment the server does not `UNSUBSCRIBE`.

When a user creates a game, the server publishes a message on the `newest` channel like so: `PUBLISH newest { "gameInfo": ... }`

In addition to that the server also does `JSON.ARRAPPEND all-games .newest {"gameInfo" : ... }` to add the game to the all games newest array, and to make sure it doesn't overflow it follows up with `JSON.arrtrim all-games .newest <begin> <end>` to shorten it back up.

Once created the games are simply retrieved with `JSON.GET game-<gameid> .`. 

Game logs are stored and retrieved in the same way, but instead of keeping track of a list of newest, there is only a pub/sub channel where open game lobbies are broadcasted like so: `PUBLISH open {"id": abc123, etc..}`.
