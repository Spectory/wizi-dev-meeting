# Wizi dev

## Offline apps overview

One of the main features of Wizitime, is the user ability to continue working while offline.
Offline architecture is more complex by default, since it adds additional logical layer at the client side.

request -> server
(request -> sync) -> server

http://wiki.genexus.com/commwiki/servlet/wiki?25536,Advanced+Concepts+of+Offline+Applications+architecture

## Multi device overview

Another main feature of the app, is the multi device support - the user may use the app with multiple devices at the same time. Each device must be aware of the other actions.

request -> server -> (sync -> db)

## Server Architecture overview
- request are passed via `Socket`
  - sockets allow message passing to multiple devices.
- request my never fail to return response. 
  - must respond with error/ok
  - no response is the same as offline from the client POV.

```Elixir
@doc """
Server may never fail to resopnd to client.
Errors are exoected to be handled by `process_handle_in`.
If for some reason, an execption unhandled, handle_in will write to log, and respond with error.
"""
def handle_in(event_name, params, socket) do
  try do
    if event_name == "ping" do
      Logger.debug("#{__MODULE__}.handle_in(#{event_name}, #{inspect params}, socket)")
    else
      Logger.info("#{__MODULE__}.handle_in(#{event_name}, #{inspect params}, socket)")
    end
    params = params
      |> Map.put("user_id", socket.assigns.user_id)
    start_process_handle_in(event_name, params, socket)
  rescue
    e ->
      Logger.error("""
      Error while processing #{__MODULE__}.handle_in
      Called with:
        #{__MODULE__}.handle_in(#{event_name}, #{inspect params}, socket)
      Caught execption:
        #{inspect e}
      """)
      {:reply, :error, socket}
  end
end

defp start_process_handle_in(event_name, params, socket) do
  case process_handle_in(event_name, params, socket) do
    :ok -> {:reply, :ok, socket}
    {:ok, response} -> {:reply, {:ok, response}, socket}
    :error -> {:reply, :error, socket}
    {:error, response} -> {:reply, {:error, response}, socket}
    :noreply -> {:noreply, socket}
  end
end
```

At the begining, the server handeld events such as `START_CLOCK`, `STOP_CLOCK`...

Why?

**hard to keep data correctness at the DB if the user supply us with invalid stream of events!**

for example [START timeEntry-A, START timeEntry-B], will lead to 2 running time entries at DB.

we deal with this by allowing only one event - the `BULK_EVENT`. `BULK_EVENT` is a group of events made by the user. when the server process it, it does so in a transaction. all or nothing.

```Elixir
defp process_handle_in(@bulk_event_name, params, socket) do
  t = Repo.transaction(fn ->
    events = params["events"]
      |> Enum.map(&(Map.put(&1, "event", Map.put(&1["event"], "user_id", socket.assigns.user_id)))) # adds user_id to each event
      |> Enum.map(&(EventHelper.create(&1["event_type"], &1["event"])))
    # This is the important part
    case EventProcessor.bulk_process(events) do
    {:ok, _res} ->
      broadcast!(
        socket,
        @need_to_update_time_entries,
        %{msg: "a `#{@bulk_event_name}` event processed"}
      )
      :ok
    {:error, _res} ->
      :error
    end
  end)
  case t do
    {:ok, _any} -> :ok
    {:error, _any} -> :error
  end
end
```

the `brodcast!` notifies all connected devices the DB state has changed, and they need to sync them selves.
notice that from the client POV, *it doesn't matter how made the changes* :)

## Client Architecture overview

The client comes out of the box with 
 - `phoenix_socket` - create socket connection and handle msg passing with PhoenixFramework.
 - `Redux` - app state management.

### Actions
Redux is action driven. state changes only by actions fired by the client.
In general actions are fired due to user actions, but at our case we need actions to be fired in the background due to incoming msgs from the socket.

### Reducers
Redux reducers handle changes to the state.

### Socket Helper
A wrapper around the `phoenix_socket` lib. its job it to handel data flow from **`Socket` to `Redux`**.

notice the usage of the `dispatcher` pattern I described at my last blog post.

```javascript
socketHelper.init = (storeDispatch) => {
  Logger.debug('socket_helper.init: initializing socket');
  socket = new Socket('/socket', { params: { token: window.userToken } });
  socket.onError(e => socketDispatcher.socketConnectionError(e));
  socket.onClose(e => socketDispatcher.socketDisconnected(e));
  socket.connect();

  channel = socket.channel(window.userChannel, {});
  channel.onError((e) => {
    socketDispatcher.channelError(e);
  });
  channel.onClose((e) => {
    socketDispatcher.channelDisconnected(e);
  });
  channel.join()
    .receive('ok', onJoinOk)
    .receive('error', onJoinError);
};

socketHelper.push = (eventName, payload, timeout) => {
  return channel.push(eventName, payload, timeout);
};
```

### SocketMiddleware
socketMiddleware coverts redux actions into outgoing socket messages.

in other words it handles the data flow from **`Redux` to the `Socket`**

```javascript
const socketMiddleware = store => next => (action) => {
  next(action); // Pass all actions through by default
  switch (action.type) {
    case SOCKET_PUSH:
      socketHelper.push(action.payload.event_type, action.payload.event, action.payload.timeout || 5000)
        .receive('error', onReceiveError(action.payload.event_type, next))
        .receive('ok', onReceiveOk(action.payload.event_type, next))
        .receive('timeout', onTimeout(action.payload.event_type, next));
      break;
    case SOCKET_BULK_PUSH:
      socketHelper.push(action.payload.event_type, action.payload.events, action.payload.timeout || 5000)
        .receive('error', onReceiveError(action.payload.event_type, next))
        .receive('ok', onReceiveOk(action.payload.event_type, next))
        .receive('timeout', onTimeout(action.payload.event_type, next));
      break;
    case GET_LATEST_TIME_ENTRIES:
      socketHelper.push(action.payload.event_type, action.payload.event, action.payload.timeout || 5000)
        .receive('error', onReceiveError(action.payload.event_type, next))
        .receive('ok', onReceiveOk(action.payload.event_type, next))
        .receive('timeout', onTimeout(action.payload.event_type, next));
      break;
    case SOCKET_RECEIVE_BROADCAST:
      handleSocketReceiveBroadcast(action.payload);
      break;
    default:
      break;
  }
};
```

### offline events

once the user performs an action (such as START_CLOCK), we need to pass this event to the server.
if the server is not available, we need to buffer the event until we have server access.

important thing to notice, is that instead of differentiate between online mode (i.e. no buffer needed), and offline mode, we can simply **always buffer**.

therefore, events that are stored on the Redux state acts as our buffer. this part of the state also persist by being stored on the local storage. the user may close the app, but the events are kept safe.

### SyncHelper
this is probably the most delicate module at the client. it is of wizi core feature - allowing the app to run while offline.

1. user performs action
2. an event is created by redux, and stored under the `state.events` (our buffer)
3. the eventReducer also triggers for sync scheduling
4. sync process starts, and locks. any future syncs requests will busy-wait on for a 15 secs.
5. sync process keeps a copy of events currently syncing (the user may perform additional actions while syncing is in progress) 
6. server response ok/error/timeout
7. response is handled by by `socketHelper`, which triggers needed Redux actions.
8. Redux reducer triggers sync cleanup
9. cleanup triggers action that removes synced events from state

```javascript
const syncAll = async () => {
  Logger.debug('syncHelper.syncAll: started!');
  let i = 0;
  while (syncingEvents) {
    if (i > SYNC_ATTEMPTS) {
      syncErrorMsg();
      syncErrorRecovery();
      return;
    }
    Logger.debug(`syncHelper.syncAll: waiting for previous sync to finish... attempt ${i}`);
    i += 1;
    await sleep(SYNC_DELAY);
  }
  const events = eventSelector.all();
  syncingEvents = [...events];
  if (!syncingEvents.length) {
    Logger.debug('syncHelper.syncAll: no events, aborting!');
    unSchedule();
    return;
  }
  const evt = bulkEvent({ events: syncingEvents });
  Logger.debug('syncHelper.syncAll: dispatching `socketBulkPush` ');
  socketDispatcher.socketBulkPush(evt);
  clearTimeout(scheduled);
  Logger.debug('syncHelper.syncAll: done!');
};

const performCleanUp = () => {
  Logger.debug('syncHelper.cleanUp: dispatching `filterOutSynced` ');
  eventDispatcher.filterOutSynced([...syncingEvents]);
  unSchedule();
};

syncHelper.schedule = (force) => {
  if (force) { clearTimeout(scheduled); }
  if (scheduled) {
    Logger.debug('syncHelper.schedule: already scheduled, rescheduling...');
    unSchedule();
    syncHelper.schedule();
  } else {
    scheduled = setTimeout(syncAll, SYNC_DELAY);
    Logger.debug(`syncHelper.schedule: scheduled sync in ${SYNC_DELAY} milliseconds`);
  }
};

syncHelper.cleanUp = () => {
  setTimeout(performCleanUp, 0);
};
```

### summery
lets see how it all plays along at the chart.

1. reacting to user action
2. reacting to socket connection.
3. reacting to server update.