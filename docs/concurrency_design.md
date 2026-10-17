# Concurrency Design

I chose Option A MultiThreading.

The server has a main listening socket thread, that spawns other worker threads
using threading.thread for each connection. I chose this so the code does need to 
use complicated selectors.


### 1 Connection Lifecycle

1. **Listen** - main thread creates a TCP socket, binds to (HOST, PORT)
2. **Accept** - main thread loops on accept(). Each call returns a new
   connection socket for that client.
3. **Register** - the server assigns the client a player number (1 or 2), and 
    sends a message with the assigned player number.
4. **Spawn worker** - the server starts a thread running
   handle_client(conn, player_num).
5. **Game loop servicing** - the worker thread loops: recv() a message,
   validates the message and acts on it accordingly
6. **Teardown** - when recv() returns an empty bytes object (client
   disconnected) or raises an exception, the worker removes that socket from
   the clients list sends message to the other player that their opponent left, and the thread returns.

Only 2 players can be in a game. If a third client connects, the server sends
ERROR Game full and closes that socket immediately. Neither of the clients use the 
main method leaving it avalible to handle errrors if anyone else tries to connect

## 2. Shared Resources and Thread Safety

One global `threading.Lock()` called `state_lock` protects all shared state:

- `board` — 6x7 grid two threads could write the same column at once
- `current_turn` — both threads read/write it to enforce turns
- `clients` — a disconnect could mutate the list mid-broadcast
- `scores` — both threads increment it when a round ends
- `game_over` — read/written at the end of a round

### Lock Acquisition Strategy

The critical section is validate -> mutate -> build message. I use
`with state_lock:` so the lock always releases.

```python
def handle_move(player_num, col):
    with state_lock:
        if player_num != current_turn:
            return "ERROR Not your turn"
        if not valid_column(col):
            return "ERROR Invalid move"

        row = drop_piece(col, player_num)
        if check_win(row, col, player_num):
            scores[player_num] += 1
            game_over = True
        else:
            switch_turn()

        msg = build_state_message()   # snapshot while locked

    broadcast(msg)                    # send outside the lock
```

Turn check and board write happen in the same lock. If the lock is released early,
player 2 could get a move in before the turn switches, causing a race
condition. Build the message while still locked, then release and send it.

## 3. Score Synchronization

Scores are kept in server memory as `scores = {1: 0, 2: 0}`.

When a round ends, the thread that handled the winning move increments the
winner's score and sets `game_over` while still holding the lock, then builds
one `GAME_OVER` message with the winner and the score table. `broadcast()`
sends that same string to both sockets, so the clients can't disagree about
the score.

## 4. Sequence Diagram

See [sequence_diagram.md](sequence_diagram.md).