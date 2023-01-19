# Creating a Real-time Chat App using Firebase and Solid Js

Solid JS is a brand new, blazingly fast framework created by @ryansolid. It takes reactivity and developer experience to a whole new level. Visually, It looks like react but is a lot better and performs a lot better.

Creating a real-time chat app using firebase and solid js will really help us understand more about solid js and how it works.

Note: This tutorial assumes you already have a firebase project setup. If you don't, follow this [guide](https://cloud.google.com/firestore/docs/client/get-firebase).

### About the Application

We are going to be creating a single page group-chatting application. Any client can go on their browser and join the chat. It is not much but will help us understand some key things about solid Js.

### Bootstrapping the Application

To bootstrap the solid.js application, we will be using the terminal command

```bash
npx degit solidjs/templates/ts firebase-chat-app
cd firebase-chat-app
```

You can go ahead and install the dependencies and run the app with your favorite package manager. I'm using yarn.

```plaintext
yarn install
yarn add firebase
yarn dev
```

**N.B:** The bootstrapped application setup looks a bit similar to react except this is a different framework: Solid.js

### Firebase Setup

To connect firebase to our application we will be using the firebase sdk.

To start off, we will be replacing `src/index.tsx` with this.

```js
import { render } from 'solid-js/web';
import App from './App';
import { initializeApp } from 'firebase/app';
import { getAuth } from 'firebase/auth';

// Include your firebase config
const firebaseConfig = {};

export const app = initializeApp(firebaseConfig);

render(() => <App />, document.getElementById('root') as HTMLElement);
```

Next, We will need to create some hooks to help us use the firebase better. We will be creating two hooks. **useAuth** and **useFirestore**

#### useAuth:

Firstly we will create a `useAuth.ts` file a new hooks directory `src/hooks/useAuth.ts`. We will then add the following to the `useAuth.ts` file:

```js
export default function useAuth(app: FirebaseApp) {
  const [authState, setAuthState] = createStore<IAuthState>({
    user: null,
    error: null,
    loading: true,
  });
  const auth = getAuth(app);
  // Setup Listeners
  auth.onAuthStateChanged(
    (user) => {
      user &&
        setAuthState(
          produce((state) => {
            state.user = user;
            state.loading = false;
          })
        );
    },
    (error) => {
      error &&
        setAuthState(
          produce((state) => {
            state.error = error;
            state.loading = false;
          })
        );
    },
    () => {
      setAuthState(
        produce((state) => {
          state.loading = false;
        })
      );
    }
  );
  // Sign in if not already signed in
  signInAnonymously(auth).then((data) => {
    setAuthState(
      produce((state) => {
        state.user = data.user;
      })
    );
  });
  return authState;
}
```

`useAuth` will enable us to get the current auth state of the person logged in. It also help us sign in anonymously if we are not already logged in.

#### useFirestore

The useFirestore hook is created to help in adding records to a firestore collection, read data(in realtime) etc.

First of all, we create a `useFirestore.ts` file in the hooks directory. The code below is what is used in the hook file:

```js
export default function useFirestore(app: FirebaseApp) {
  const db = getFirestore(app);
  const authState = useAuth(app);
  async function addData(collectionName: string, data: Record<string, any>) {
    const doc = await addDoc(collection(db, collectionName), {
      ...data,
      from: authState?.user?.uid,
      created: Timestamp.now(),
    });
    console.log('doc', doc);
  }

  function getData(collectionName: string) {
    const [data, setData] = createSignal<DocumentData[]>([]);
    getDocs(collection(db, collectionName)).then((records) => {
      setData(records.docs.map((doc) => doc.data()));
    });

    const q = query(collection(db, collectionName), orderBy('created', 'asc'));
    onSnapshot(q, (snapshot) => {
      setData(snapshot.docs.map((doc) => doc.data()));
    });
    return data;
  }

  return { db, addData, getData };
}
```

As you can see, we are creating two functions: One to add data to a given collection and another to get data from a specified collection. The addData function also sets up a listener to firestore to get the chats in realtime.

### Building the app

Before We dive into making realtime chat work, we will create the ui for this using solid js.

---

First of all, we will have only one page which is the chat screen page. This is going to be in the App component. The main div wraps the title, messages and message input box.

```js
const App: Component = () => {
  const [messageTerm, setMessageTerm] = createSignal('');
  const [messageId, setMessageId] = createSignal('vision');

  return (
    <div class={styles.App}>
      <h1>Message the Group</h1>
      <div class={styles.App__messages}>
        <For each={messages()}>
          {(message) => (
            <div
              class={
                message.from == messageId()
                  ? styles.App__messageLocal
                  : styles.App__messageForeign
              }
            >
              {message.message}
            </div>
          )}
        </For>
      </div>
      <div class={styles.App__input}>
        <textarea
          placeholder='Message the group'
          onKeyUp={(e) => setMessageTerm(e.currentTarget.value)}
        />
        <button class={styles.App__button}>Send</button>
      </div>
    </div>
  );
};
```

There are a couple of things to notice about the solid.js code and its difference to react.

1. **Usage of For Component instead of Mapping**
    
2. **Signals**
    
3. **Lifecycle hooks**
    

Among others.

### Putting it all together

Currently the messages are hardcoded and the send button does nothing. This next step is to add firebase functionality to our chat application.

Firstly, We will need to extract the chat into a new component.

* Create a new file `src/components/Chat.tsx`. Add the chat screen code into it that was previously in App component.
    
* Make sure imports are pointing to the right place.
    

Now that the chat is in a new file, we can replace `App.tsx` with

```js
const App: Component = () => {
  const authState = useAuth(app);

  return (
    <Switch>
      <Match when={authState.loading}>
        <p>Loading...</p>
      </Match>
      <Match when={authState.error}>
        <p>Error</p>
      </Match>
      <Match when={authState.user}>
        <Chat />
      </Match>
    </Switch>
  );
};
```

The app component now uses our custom `useAuth` hook to listen to the user's current auth state and renders a loading, error text or the chat screen based on the auth state.

**Note:** The Switch component is used for conditional rendering in solid when there are multiple conditions.

---

Next up, we will add firestore to our chat component. To achieve this we need to do the following:

Firstly, we get the data from firestore using our custom hooks:

```js
const { addData, getData } = useFirestore(app);
const data = getData('chats');
const authState = useAuth(app);
```

The first two lines are for getting the data while the third is for the getting the current auth state.

Next, we make the messages dynamic. Previously the messages were hard coded but since we now have the firestore messages, we can replace the hardcoded messages with realtime messages as follows:

```jsx
<For each={data()}>
  {(message) => (
    <div
     class={
       message.from == authState?.user?.uid
         ? styles.App__messageLocal
         : styles.App__messageForeign
      }
    >
      {message.message}
    </div>
   )}
</For>
```

Two things are happening in the code block above. Firstly we are replacing hardcoded messages array with the actual data. Secondly, the message source is gotten by comparing the message(firestore) from field with our unique id.

Finally, we can add an onClick handler to our send button to send the messages. The button with the onclick handler is now

```js
<button class={styles.App__button} onClick={handleSubmit}>
   Send
</button>
```

The handleSubmit function implementation just adds sends the message to firestore. It is implemented like this:

```js
function handleSubmit() {
  addData('chats', {
    message: messageTerm(),
  });
  submit_input.value = '';
}
```

We are using the addData function gotten from our useFirestore hook. `submit_input` is a ref to our textarea.

After doing all these, our website now looks like(with my styles)

![Chat Screen](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/k4em95i3zjht8vteck3v.png align="left")

You can test it by opening two different browser clients and chatting.

### Conclusion

Creating this project has really helped us to understand solidjs concepts more like stores, signals, produce etc.

The deployed version can be found on [https://realtime-solid-chat-app.web.app/](https://realtime-solid-chat-app.web.app/). All the source code can be found on [github](https://github.com/psecuresystem/Solidjs-Firebase-Realtime-chat-App).

Thanks. [Buy me a coffee](https://www.buymeacoffee.com/visiondaniels32)