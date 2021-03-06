## Long running services

Most *signals* in your application are triggered by users interacting with the UI-- typing in an input, submitting a form, or closing a modal window. But, many times your application needs to respond to events from the "outside" world. These could be websocket messages from services like Firebase, chat messages from your backend, or even simple HTTP long polling for data.

In Cerebral, these processes are encapsulated using the *module* interface and are exposed as *services* to your actions. Let's walk through an example HTTP long polling service.

#### An HTTP Polling Module

Imagine you have a Navigation bar at the top of your app that needs to display the unread message count for the current user. It queries an **/api/message_counts/:id** endpoint every 10 seconds to fetch the current **newMessageCount** and display in a component.

First, let's set up a skeleton with the expected *module* interface. This hooks our code into Cerebral by registering **state**, **services** and **signals**. It also gives us access to the controller which allows us to extract information about it inside our services to, for example to trigger a signal.

```js
// ./modules/http-poller/index.js

export default (module, controller) => {
}
```

The *module* interface is a function that is passed the **module** instance and the cerebral **controller** as arguments.

Modules hook into the **addModules** method of the controller:

```js
// ./main.js

import {Controller} from 'cerebral';
import Model from 'cerebral/models/immutable';
import HttpPoller from './modules/http-poller';
import axios from 'axios';

const controller = Controller(Model({}))

controller.addServices({
  ajax: axios
});

controller.addModules({
  httpPoller
});
```

This makes the module available in your actions:

```js
// ./navbar/actions/poll-message-counts.js

function pollMessageCounts({state, services}) {
  const {httpPoller} = services;

  // do cool stuff when we implement the httpPoller
}
```

#### Implementation

Now that we have the basic pieces in place, how do we actually wire all this up?

Let's write it out:

- When the Navbar is mounted, we should trigger a **mounted** signal
- The **mounted** signal should call a **pollMessageCounts** action
- The **pollMessageCounts** action should call our **/message_counts** API every 10 seconds for the new message count.
- On *success* we should trigger a *new* **messageCountsFetched** signal
- The **messageCountsFetched** signal should take care of setting the new counts in the state tree.

Nice! Let's implement the `httpPoller`:

```js

const TEN_SECONDS = 10000;

export default (module, controller) => {

  const ajax = controller.getServices('ajax');

  module.addServices({
    atInterval: (url, signals, interval = TEN_SECONDS) => {
      const {success, error} = signals;

      return window.setInterval(() => {
        ajax.get(url)
          .then(response => {
            controller.getSignals(success)({
              data: response.data
            });
          })
          .catch(response => {
            controller.getSignals(error)({response});
          });
      }, interval);
    }
  });
}

```
**httpPoller** uses the **controller.addServices()** function to expose an **atInterval** method to the **services** argument in your actions. **atInterval** takes a **url** to poll and an object defining **success** and **error** signal names.

Using **window.setInterval** we query our endpoint once every 10 seconds. In the **then()** callback we use the **controller.getSignals()** method to get our signal by name and call it with the returned **message_counts** data. Same goes for our **catch()** callback.

#### Defining the Action

Lets use the **httpPoller** service in our action:

```js
function pollMessageCounts({state, services}) {
  const {httpPoller} = services;
  const id = state.get('currentUser.id');

  httpPoller.atInterval(`/api/message_counts/${id}`, {
    success: 'navbar.messageCountsFetched',
    error: 'navbar.errorFetchingMessageCounts'
  })
}
```

We grab our **httpPoller** service from the **services** key passed to our action. Then, we pass in *success* and *error* signals to be triggered every 10 seconds.

#### Tying it Together with Signals

Now we can implement our signals and their corresponding action chains in our **Navbar** module and define our component:

```js
// ./modules/navbar/index.js

export default module => {

  module.addSignals({
    mounted: [
      pollMessageCounts
    ],
    messageCountsFetched: [
      copy('input:data', 'state:navbar.messageCounts')
    ],
    errorFetchingMessageCounts: [
      // handle error here
    ]
  })
}
```

Then we can display the **messageCounts** in our component:

```js
import React,{Component} from 'react';
import {connect} from 'cerebral-view-react';

export default connect({
  messageCounts: 'navbar.messageCounts'
},
  function NewMessagesIndicator(props) {
    const newCount = props.messageCounts.new_count

    return (
      <div>{newCount || '--'}</div>
    )
  }
)
```

And we're done!

#### Summary

- Cerebral's *module* interface allows you to create a service (process) that gives input to Cerebral by firing off signals
- Services are accessible in the *actions*, so at any point in a flow you can control the behaviour of a service. Start and stop subscriptions, open and close connections etc.
- Unlike common ajax calls that has only one response, returning a promise, you may use the approach explained here to handle processes that will give multiple responses over time
- Since all inputs to your app are explicitly listed in your signal definition you can easily understand their behavior and view these processes in the debugger
