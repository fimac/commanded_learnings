# Commanded Learnings

Commanded is an Elixir library to build applications that require event sourcing. It follows the CQRS (Command Read Responsibility Segregation) / ES (Event Sourcing) pattern.
Where reads and write logic are separated.

Some resources I used to learn about Event Sourcing:
- `https://www.youtube.com/watch?v=JHGkaShoyNs` - Greg Young talk on Event Sourcing and CQRS 
- `https://www.youtube.com/watch?v=S3f6sAXa3-c` - Bernardo Amorim - CQRS and Event Sourcing - Code Beam SF 2018
- `https://martinfowler.com/eaaDev/EventSourcing.html` - Martin Fowler - Event Sourcing


To work out how to implement ES/CQRS in Elixir I'm using the bank account example (`git@github.com:fimac/event_sourcing_bank_api.git`) that 
I built following this tutorial `https://blog.nootch.net/post/event-sourcing-with-elixir-part-1/`.

There are IO.inspects added in the flow from the router(web) > controller > context > router(commanded) > validation > aggregate > projector to work out what's being
called when and learn more about what Commanded is doing.

Outcomes:
- Learn more about Event Sourcing and CQRS via the implementation of it.
- To learn how much commanded abstracts away.
- How do the execute and apply functions get called in the aggregate account module.
- When and where do event's get persisted.

I cloned the below repos and started by searching for the logger.debug messages from the output in the console to work out which part of the source code to look for.
- `git@github.com:commanded/commanded.git`
- `git@github.com:commanded/eventstore.git`
- `git@github.com:commanded/commanded-ecto-projections.git`

### CQRS/ES notes

Commands
- A request
- Imperative
- e.g Add funds, open bank account

Events
- A fact
- In the past
- e.g FundsAdded, bank account opened

Aggregates
- Like the glue
- enforce business invariants, I need to maintain consistency, make sure the account doesn't go overdrawn
- has some state
- state changes with events
- may emit events in response to commands.

Projectors
- 


### Breaking down the output from the console.

The full output from the console is in this text file [console output](console_output.txt)


1. We make a post call to a route we have set up at /api/accounts that calls `account_controller.create/2`.
  
`curl -i -X POST -H 'Content-Type:application/json' -d '{"account": {"initial_balance":1000 } }' http://localhost:4000/api/accounts`

```
iex(4)> [info] POST /api/accounts
[debug] Processing with BankAPIWeb.AccountController.create/2
  Parameters: %{"account" => %{"initial_balance" => 1000}}
  Pipelines: [:api]
```

2. Create then calls a function in the accounts context which starts the flow of creating a command, event, persisting that event, updating the read modal.
   `account_controller.create/2` calls the open account function in the accounts context `BankAPI.Accounts.open_account/1`

    Open account handles the dispatch call to send the command to the aggregate, and the response. The BankAPI.Router.dispatch() call is made, which is a `CommandedCommanded.Commands.Router` macro. The command (`%OpenAccount{}` - fields :initial_balance, :account_uuid) is dispatch to the aggregate.

3. In the router there is validation on the command struct, using another `Commanded.Commands.Router` macro, `middleware/1` , which validates the fields in the
   command struct, in the account example it's checking that the command has a positive integer and checks the account uuid against a regex.

   [Middleware docs](https://github.com/commanded/commanded/blob/04afe01ed51076db73ba90298425357ed4d2af18/lib/commanded/commands/router.ex#L241-L242)

4. There are 2 ways to dispatch the command this can be done via a command handler, or send it directly to the aggregate. In this example we are sending the command directly to the aggregate. (Q: Whats the difference? Why would you do one of the other)

Looking at the source code, dispatch calls a private function called parse_opts.

[Dispatch and parse opts](https://github.com/commanded/commanded/blob/04afe01ed51076db73ba90298425357ed4d2af18/lib/commanded/commands/router.ex#L626-L627)

One of the questions I had was how does the execute and apply functions in the aggregate module `BankAPI.Accounts.Aggregates.Account` get called. 

In our case we are sending our command directly to the aggregate, on line :630 I think is where we can see that if we send it directly to the aggregate, then it expects the aggregate to have an execute function. If we're sending it to a handler, then it expects that module to have a handle function.

[Commanded.Commands.Router docs](BankAPI.Accounts.Aggregates.Account)


There's a bunch of stuff I don't understand in the source code at this point, there's a dispatch callback, which has some macro's and looks like it's creates a payload, this payload then is used in another function called dispatch in the module Dispatcher.

`Dispatcher.dispatch -> calls Commanded.Aggregates.Supervisor.open_aggregate/3`

This then creates a genserver process. From the docs: `Open an aggregate instance process for the given aggregate module and unique identity.`

```
[debug] Locating aggregate process for `BankAPI.Accounts.Aggregates.Account` with UUID "170a9c58-2b1b-41e3-bed1-b6db5f4942a1" <---- this is the logger from the open aggregate function.
```

`Commanded.Aggregates.Aggregate`

Aggregate is a GenServer process used to provide access to aninstance of an event sourced aggregate.
It allows execution of commands against an aggregate instance, and handles persistence of created events to the configured event store. Concurrent commands sent to an aggregate instance are serialized and executed in the order received.

Aggregate.execute is called, which then does a GenServer.call() to :execute_command. The handle_call callback calls the function execute_command, this then calls a function called apply_and_persist_events.

```
[debug] BankAPI.Accounts.Aggregates.Account<170a9c58-2b1b-41e3-bed1-b6db5f4942a1@0> executing command: %BankAPI.Accounts.Commands.OpenAccount{account_uuid: "170a9c58-2b1b-41e3-bed1-b6db5f4942a1", initial_balance: 1000}  <---- defp execute_command being called.

aggregates account execute <----- This IO puts is in the return of the execute function.

aggregates account - apply <------ Apply function is called in aggregate -> This creates a new Account struct with the updated data i.e balance in this example.
```

Within the apply_and_persist_events function, there is a function called append_to_stream which called EventStore.append_to_stream, THIS is where EventStore comes in.

# Event Store functions

`EventStore.append_to_stream > EventStore.Streams.Stream.append_to_stream > Stream.prepare_stream > Storage.create_stream > CreateStream.execute`

So I think when the event get's appended to the stream, this is when the event is persisted to the postgres db event store.

```
[debug] Attempting to create stream 170a9c58-2b1b-41e3-bed1-b6db5f4942a1 <-- CreateStream.execute
[debug] Created stream "170a9c58-2b1b-41e3-bed1-b6db5f4942a1" (id: 24) <- CreateStream.handle_response

[debug] Appended 1 event(s) to stream "170a9c58-2b1b-41e3-bed1-b6db5f4942a1" <-- Storage.Appender.ex -- event added to the stream. This must be where the data is persisted. Because below the received events has a RecordedEvent struct, which is the persisted data and metadata for a single event

[debug] Listener received notification on channel "events" with payload: "170a9c58-2b1b-41e3-bed1-b6db5f4942a1,24,1,1"

[info] Sent 201 in 9ms
[debug] Listener received notification on channel "events" with payload: "$all,0,32,32" <---- EventStore.Notifications.Listener
```

`EventStore.RecordedEvent` contains the persisted data and metadata for a single event.

```
[debug] BankAPI.Accounts.Aggregates.Account<170a9c58-2b1b-41e3-bed1-b6db5f4942a1@1> received events: [%Commanded.EventStore.RecordedEvent{causation_id: "a045e247-8cc8-4025-9258-900524690f0c", correlation_id: "2995efce-ab0a-463c-8c77-1488ca7b5428", created_at: ~N[2020-04-30 00:12:16.173110], data: %BankAPI.Accounts.Events.AccountOpened{account_uuid: "170a9c58-2b1b-41e3-bed1-b6db5f4942a1", initial_balance: 1000}, event_id: "476cbe10-4699-477d-bea6-9e9f13d6e67d", event_number: 1, event_type: "Elixir.BankAPI.Accounts.Events.AccountOpened", metadata: %{}, stream_id: "170a9c58-2b1b-41e3-bed1-b6db5f4942a1", stream_version: 1}]
[debug] Subscription "Accounts.Projectors.AccountClosed"@"$all" received 1 event(s)
[debug] Subscription "Accounts.Projectors.DepositsAndWithdrawals"@"$all" received 1 event(s)
[debug] Subscription "Accounts.Projectors.AccountOpened"@"$all" received 1 event(s)
[debug] Subscription "Accounts.Projectors.AccountClosed"@"$all" is enqueueing 1 event(s)
[debug] Subscription "Accounts.Projectors.AccountOpened"@"$all" is enqueueing 1 event(s)
[debug] Subscription "Accounts.Projectors.DepositsAndWithdrawals"@"$all" is enqueueing 1 event(s)
[debug] Subscription "Accounts.ProcessManagers.TransferMoney"@"$all" received 1 event(s)
[debug] Subscription "Accounts.ProcessManagers.TransferMoney"@"$all" is enqueueing 1 event(s)
```

The below output is from the multi function in BankAPI.Accounts.Projectors.AccountOpened.

```
event *******: %BankAPI.Accounts.Events.AccountOpened{
  account_uuid: "170a9c58-2b1b-41e3-bed1-b6db5f4942a1",
  initial_balance: 1000
}
[debug] BankAPI.Accounts.ProcessManagers.TransferMoney received 1 event(s)
[debug] BankAPI.Accounts.ProcessManagers.TransferMoney is not interested in event 32 ("170a9c58-2b1b-41e3-bed1-b6db5f4942a1"@1)
[debug] BankAPI.Accounts.Projectors.AccountOpened received events: [%Commanded.EventStore.RecordedEvent{causation_id: "a045e247-8cc8-4025-9258-900524690f0c", correlation_id: "2995efce-ab0a-463c-8c77-1488ca7b5428", created_at: ~N[2020-04-30 00:12:16.173110], data: %BankAPI.Accounts.Events.AccountOpened{account_uuid: "170a9c58-2b1b-41e3-bed1-b6db5f4942a1", initial_balance: 1000}, event_id: "476cbe10-4699-477d-bea6-9e9f13d6e67d", event_number: 32, event_type: "Elixir.BankAPI.Accounts.Events.AccountOpened", metadata: %{}, stream_id: "170a9c58-2b1b-41e3-bed1-b6db5f4942a1", stream_version: 1}]
[debug] BankAPI.Accounts.ProcessManagers.TransferMoney confirming receipt of event: 32
[debug] BankAPI.Accounts.Projectors.AccountClosed received events: [%Commanded.EventStore.RecordedEvent{causation_id: "a045e247-8cc8-4025-9258-900524690f0c", correlation_id: "2995efce-ab0a-463c-8c77-1488ca7b5428", created_at: ~N[2020-04-30 00:12:16.173110], data: %BankAPI.Accounts.Events.AccountOpened{account_uuid: "170a9c58-2b1b-41e3-bed1-b6db5f4942a1", initial_balance: 1000}, event_id: "476cbe10-4699-477d-bea6-9e9f13d6e67d", event_number: 32, event_type: "Elixir.BankAPI.Accounts.Events.AccountOpened", metadata: %{}, stream_id: "170a9c58-2b1b-41e3-bed1-b6db5f4942a1", stream_version: 1}]
[debug] BankAPI.Accounts.Projectors.AccountClosed confirming receipt of event #32
[debug] BankAPI.Accounts.Projectors.DepositsAndWithdrawals received events: [%Commanded.EventStore.RecordedEvent{causation_id: "a045e247-8cc8-4025-9258-900524690f0c", correlation_id: "2995efce-ab0a-463c-8c77-1488ca7b5428", created_at: ~N[2020-04-30 00:12:16.173110], data: %BankAPI.Accounts.Events.AccountOpened{account_uuid: "170a9c58-2b1b-41e3-bed1-b6db5f4942a1", initial_balance: 1000}, event_id: "476cbe10-4699-477d-bea6-9e9f13d6e67d", event_number: 32, event_type: "Elixir.BankAPI.Accounts.Events.AccountOpened", metadata: %{}, stream_id: "170a9c58-2b1b-41e3-bed1-b6db5f4942a1", stream_version: 1}]
[debug] BankAPI.Accounts.Projectors.DepositsAndWithdrawals confirming receipt of event #32
```

Projection version is updated for account opened
```
multi &&&&&&&&&: %Ecto.Multi{
  names: #MapSet<[:projection_version, :verify_projection_version]>,
  operations: [
    projection_version: {:changeset,
     #Ecto.Changeset<
       action: :update,
       changes: %{last_seen_event_number: 32},
       errors: [],
       data: #BankAPI.Accounts.Projectors.AccountOpened.ProjectionVersion<>,
       valid?: true
     >, [prefix: nil]},
    verify_projection_version: {:run,
     #Function<1.79214068/2 in BankAPI.Accounts.Projectors.AccountOpened.update_projection/3>}
  ]
}
account opened projector: %Ecto.Multi{
  names: #MapSet<[:account_opened, :projection_version,
   :verify_projection_version]>,
  operations: [
    account_opened: {:changeset,
     #Ecto.Changeset<action: :insert, changes: %{}, errors: [],
      data: #BankAPI.Accounts.Projections.Account<>, valid?: true>, []},
    projection_version: {:changeset,
     #Ecto.Changeset<
       action: :update,
       changes: %{last_seen_event_number: 32},
       errors: [],
       data: #BankAPI.Accounts.Projectors.AccountOpened.ProjectionVersion<>,
       valid?: true
     >, [prefix: nil]},
    verify_projection_version: {:run,
     #Function<1.79214068/2 in BankAPI.Accounts.Projectors.AccountOpened.update_projection/3>}
  ]
}
[debug] QUERY OK db=0.3ms idle=1464.9ms
begin []
[debug] QUERY OK source="projection_versions" db=2.7ms
SELECT p0."projection_name", p0."last_seen_event_number", p0."inserted_at", p0."updated_at" FROM "projection_versions" AS p0 WHERE (p0."projection_name" = $1) ["Accounts.Projectors.AccountOpened"]
[debug] QUERY OK db=0.4ms
UPDATE "projection_versions" SET "last_seen_event_number" = $1, "updated_at" = $2 WHERE "projection_name" = $3 [32, ~N[2020-04-30 00:12:16], "Accounts.Projectors.AccountOpened"]
[debug] QUERY OK db=0.5ms
INSERT INTO "accounts" ("current_balance","status","uuid","inserted_at","updated_at") VALUES ($1,$2,$3,$4,$5) [1000, "open", <<23, 10, 156, 88, 43, 27, 65, 227, 190, 209, 182, 219, 95, 73, 66, 161>>, ~N[2020-04-30 00:12:16], ~N[2020-04-30 00:12:16]]
[debug] QUERY OK db=0.4ms
commit []
[debug] BankAPI.Accounts.Projectors.AccountOpened confirming receipt of event #32
```