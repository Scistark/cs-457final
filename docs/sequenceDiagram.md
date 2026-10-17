# Sequence Diagram

How a move gets validated and broadcast to both clients.

```mermaid
sequenceDiagram
    autonumber
    actor Client1 as Client 1 (X)
    participant Server as Game Server
    actor Client2 as Client 2 (0)

    Client1->>Server: MOVE col=3
    Note over Server: Worker thread for Client 1<br>takes state_lock
    alt Valid move and Client 1's turn
        Note over Server: Drop piece, check for win
        alt No winner yet
            Note over Server: current_turn = 2<br>release lock
            Server-->>Client1: STATE_UPDATE board, turn=2
            Server-->>Client2: STATE_UPDATE board, turn=2
        else Client 1 got four in a row
            Note over Server: scores[1] += 1<br>game_over = True<br>release lock
            Server-->>Client1: GAME_OVER winner=1, scores
            Server-->>Client2: GAME_OVER winner=1, scores
        end
    else Not their turn, or column full
        Note over Server: Board unchanged<br>release lock
        Server-->>Client1: ERROR Not your turn
    end
```

## Notes

Client 1 sends the column it wants to drop into. The client doesnt do any error checking that is done at the server.

The server checks that it is that cleints turn and that their placment if their piece is valid within bounds and has propper space.

Once the move is validated the server drops the piece into the lowest avalible place in that column, and also checks to see if it is a win. If a palayer won the score of that player is updated, if no win occurs the wserver updates the current_turn and sends updated state message out. 


So a single message is sent out to both clients with an upadted board and score.