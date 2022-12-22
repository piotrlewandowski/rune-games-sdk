---
sidebar_position: 2
---

# Multiplayer SDK

:::caution

The Multiplayer SDK is currently in beta. There will be some rough edges, and some APIs might change.

:::

Rune methods for writing multiplayer games. The SDK is split into two parts:

- **Game logic**, this code needs to be located in a file called `logic.js`
- **UI integration**, that can live anywhere, but the docs will refer to `client.js`

## `Rune.initLogic(options)`

The `initLogic` function should be called directly at the top level of your `logic.js` file. This should contain all logic required to control your game state and handle player life cycle events. All options except `events` are required. Example:

```js
// logic.js
Rune.initLogic({
  minPlayers: 1,
  maxPlayers: 4,
  setup: (players) => {
    const scores = {}
    for (playerId in players) {
      scores[playerId] = 0
    }
    return { scores }
  },
  actions: {
    myAction: (payload, { game, playerId }) => {
      // Check it's not the other player's turn
      if (game.lastPlayerTurn !== playerId) {
        throw Rune.invalidAction()
      }

      // Increase score and switch turn
      game.scores[playerId]++
      game.lastPlayerTurn = playerId

      // Determine if game has ended
      if (isVictoryOrDraw(game)) {
        Rune.gameOver()
      }
    },
  },
  events: {
    playerJoined: (playerId, { game }) => {
      game.scores[playerId] = 0
    },
    playerLeft: (playerId, { game }) => {
      delete game.scores[playerId]
    },
  },
})
```

### `minPlayers: number`

A value between 1-4 of the minmum amount of players that is required to play the game. See [Joining and Leaving](multiplayer/joining-leaving.md#minimum-and-maximum-players).

### `maxPlayers: number`

A value between 1-4, must be equal to or greater than `minPlayers`. If the value is lower than 4, other users may join the game as spectators. See [Joining and Leaving](multiplayer/joining-leaving.md#minimum-and-maximum-players).

### `setup(playerIds: string[]): any`

The `setup` function returns the initial values for the game state, which is the game information that’s synced across players. The function gets the `players` argument which is an array of the player IDs at the time of starting the game and must return the initial game state.

### `actions: { [string]: (payload, { game: any, playerId: string }) => void}`

The `actions` option is an object with actions functions exposed to the UI integration layer, called with [`Rune.actions.myAction(payload)`](#runeactionspayload). The functions are responsible for [validating the action](#runeinvalidaction), mutating the `game` state and [end the game](#runegameover) when appropriate.

### `events: { playerJoined? | playerLeft?: (playerId: string, { game: any }) => void }` _optional_

By default a game will end if a player leaves (see [Joining and Leaving](multiplayer/joining-leaving.md#minimum-and-maximum-players)), but by defining the `playerJoined`/`playerLeft` events you can [Support Players Joining Midgame](multiplayer/joining-leaving.md#supporting-players-joining-midgame).

## `Rune.invalidAction()`

Whenever a player tries to do an action that is not allowed, the action handler should reject it by calling `throw Rune.invalidAction()` which will cancel the action and potentially roll back optimistic updates.

```js
// logic.js
Rune.initLogic({
  actions: {
    myAction: (payload, { game, playerId }) => {
      if (!isValidAction(payload)) {
        throw Rune.invalidAction()
      }
    },
  },
})
```

## `Rune.gameOver()`

When the game has ended (with a winner or draw), the action handler should call `Rune.gameOver()`. Your game doesn't need to show a "game over" screen. Rune overlays a standardized high score interface to the user.

```js
// logic.js
Rune.initLogic({
  actions: {
    myAction: (payload, { game, playerId }) => {
      if (isVictoryOrDraw(game)) {
        Rune.gameOver()
      }
    },
  },
})
```

## `Rune.initClient(options)`

The `initClient` function should be called after your game is fully ready, but should not start the actual gameplay until `visualUpdate` is called.

```js
// client.js
Rune.initClient({
  visualUpdate: ({
    newGame,
    oldGame,
    yourPlayerId,
    players,
    action,
    event,
    rollbacks,
  }) => {
    render(newGame)
  },
})
```

### `visualUpdate: () => void`

#### `newGame: any`

This argument is the current game state. Your `visualUpdate()` function should update the UI to reflect its values.

#### `oldGame: any`

This argument is the previous game state. Usually this can be ignored, but it's useful if your game needs to detect changes of certain values.

#### `yourPlayerId?: string`

Your player id, if the current user is a spectator this argument is undefined.

#### `players: Record<string, { playerId: string, displayName: string }>`

The `players` argument is an object of the current players, useful to display their names in the game.

:::caution

Do not rely on `players` argument for determining player order, instead use `setup` and `events.playerJoined/playerLeft` callbacks in `logic.js`.

:::

#### `action?: { action: string, playerId: string, params: any }`

If the update was triggered from a `Rune.actions.*` call, this argument will contain info about it, such as the payload and who initiated. Usually this should be ignored and rely on `newGame` instead.

#### `event?: { event: string, params: any }`

Currently this is always the `stateSync` event, you can ignore it for now.

## `Rune.actions.*(payload)`

All functions passed in the `actions` object in `Rune.initLogic()` will be exposed to the client via `Rune.actions.myActionName`. This is the only way game state may be updated to make sure it's propagated to every player. You may call it with one argument of any type, but usually an object is recommended.

```js
// client.js
button.onClick = () => {
  Rune.actions.markCell({
    myId: "button",
  })
}
```