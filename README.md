# commanded_learnings

iex(4)> [info] POST /api/accounts <----------- Post call made to accounts url

[debug] Processing with BankAPIWeb.AccountController.create/2
  Parameters: %{"account" => %{"initial_balance" => 1000}}
  Pipelines: [:api]

account controller create: %{"initial_balance" => 1000} <------- Hits accounts controller create function. Which then calls the open account function in the accounts context.


accounts context - open account function <----- BankAPI.Accounts.open_account function, which then calls BankAPI.Router.dispatch()

######
BankAPI.Router, which uses the Commanded.Commands.Router module.
the dispatch macro is called, and the command is sent directly to the aggregate (BankAPI.Accounts.Aggregates.Account)

https://github.com/commanded/commanded/blob/04afe01ed51076db73ba90298425357ed4d2af18/lib/commanded/commands/router.ex#L349-L350

Before the dispatch macro is called, there is a middleware macro that is called that is using skoom that validates the command. In this case it checks the inital balance is positive, and there's a regex
that checks the uuid.

https://github.com/commanded/commanded/blob/04afe01ed51076db73ba90298425357ed4d2af18/lib/commanded/commands/router.ex#L241-L242

commands - open account, validate command: %BankAPI.Accounts.Commands.OpenAccount{  <------- command is validated BankAPI.Accounts.Commands.OpenAccount
  account_uuid: "170a9c58-2b1b-41e3-bed1-b6db5f4942a1",
  initial_balance: 1000
}

The dispatch macro is called, in this example we are sending the command directly to the aggregate. This calls a private function called parse_opts, within here it shows there are 2 options. In our
case we are sending our command directly to the aggregate, this is where we can see how it's expecting the execute function to the aggregate module. If we're sending it to a command handler, then it 
expects that module to have a handle function.

https://github.com/commanded/commanded/blob/04afe01ed51076db73ba90298425357ed4d2af18/lib/commanded/commands/router.ex#L626-L627


[debug] Locating aggregate process for `BankAPI.Accounts.Aggregates.Account` with UUID "170a9c58-2b1b-41e3-bed1-b6db5f4942a1"
[debug] BankAPI.Accounts.Aggregates.Account<170a9c58-2b1b-41e3-bed1-b6db5f4942a1@0> executing command: %BankAPI.Accounts.Commands.OpenAccount{account_uuid: "170a9c58-2b1b-41e3-bed1-b6db5f4942a1", initial_balance: 1000}

aggregates account execute <----- Execute function is called in aggregate -> This returns an event
aggregates account - apply <------ Apply function is called in aggregate -> This creates a new Account struct with the updated data i.e balance in this example.

[debug] Attempting to create stream 170a9c58-2b1b-41e3-bed1-b6db5f4942a1
[debug] Created stream "170a9c58-2b1b-41e3-bed1-b6db5f4942a1" (id: 24)
[debug] Appended 1 event(s) to stream "170a9c58-2b1b-41e3-bed1-b6db5f4942a1"
[debug] Listener received notification on channel "events" with payload: "170a9c58-2b1b-41e3-bed1-b6db5f4942a1,24,1,1"
[info] Sent 201 in 9ms
[debug] Listener received notification on channel "events" with payload: "$all,0,32,32"
[debug] BankAPI.Accounts.Aggregates.Account<170a9c58-2b1b-41e3-bed1-b6db5f4942a1@1> received events: [%Commanded.EventStore.RecordedEvent{causation_id: "a045e247-8cc8-4025-9258-900524690f0c", correlation_id: "2995efce-ab0a-463c-8c77-1488ca7b5428", created_at: ~N[2020-04-30 00:12:16.173110], data: %BankAPI.Accounts.Events.AccountOpened{account_uuid: "170a9c58-2b1b-41e3-bed1-b6db5f4942a1", initial_balance: 1000}, event_id: "476cbe10-4699-477d-bea6-9e9f13d6e67d", event_number: 1, event_type: "Elixir.BankAPI.Accounts.Events.AccountOpened", metadata: %{}, stream_id: "170a9c58-2b1b-41e3-bed1-b6db5f4942a1", stream_version: 1}]
[debug] Subscription "Accounts.Projectors.AccountClosed"@"$all" received 1 event(s)
[debug] Subscription "Accounts.Projectors.DepositsAndWithdrawals"@"$all" received 1 event(s)
[debug] Subscription "Accounts.Projectors.AccountOpened"@"$all" received 1 event(s)
[debug] Subscription "Accounts.Projectors.AccountClosed"@"$all" is enqueueing 1 event(s)
[debug] Subscription "Accounts.Projectors.AccountOpened"@"$all" is enqueueing 1 event(s)
[debug] Subscription "Accounts.Projectors.DepositsAndWithdrawals"@"$all" is enqueueing 1 event(s)
[debug] Subscription "Accounts.ProcessManagers.TransferMoney"@"$all" received 1 event(s)
[debug] Subscription "Accounts.ProcessManagers.TransferMoney"@"$all" is enqueueing 1 event(s)
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