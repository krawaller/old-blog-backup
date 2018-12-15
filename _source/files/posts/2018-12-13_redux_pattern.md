---
type: post
title: "How to reduce boilerplate in Redux"
date: 2018-12-13
tags: [redux, typescript, cyclejs]
author: David
excerpt: Exploring a pattern that lets us group related logic and reduce boilerplate in Redux apps
---

### Premise

[Redux](https://redux.js.org/) is frequently bemoaned for forcing you to write a fair bit of boilerplate. Sometimes that critisism is not fair as the code in question is actually necessary, but definitely Redux app can often feel very verbose.

In October I attended [CycleConf](http://cycleconf.com/), the yearly conference for the [CycleJS](https://cycle.js.org/) framework. There I got to play with [Cycle State](https://cycle.js.org/api/state.html), a new API where each component gets the current state as an input stream, and returns a stream of reducers to calculate the next state.

CycleJS is of course very (very) different from a Redux-driven app, but I realised that the same pattern can be applied to Redux nonetheless! And doing so means less boilerplate and better grouping of related logic.

In this post we'll explore a TypeScript-driven example, where we'll first write "traditional" Redux code, and then refactor it to the pattern in question.

For brevity's sake I'll be cutting some corners, so neither logic nor modelling will be optimal with regards to perfomance and convenience.

### The data model

Imagine that we have an app with _users_ and _posts_:

```typescript
type User = {
  name: string;
  postIds: string[];
};

type Post = {
  title: string;
  content: string;
  authorId: string;
};
```

They each live in a top-level prop in our _app state_:

```typescript
type UserState = { [ID: string]: User };
type PostState = { [ID: string]: Post };

type AppState = {
  users: UserState;
  posts: PostState;
  // ...probably more stuff here too
};
```

### Creating actions

We'll be using the popular action format where all additional data are collected in a `payload` prop:

```typescript
type ActionBlueprint<T, P> = {
  type: T;
  payload: P;
};
```

For our experiment, let's make up two actions - we want to _delete posts_ and _add posts_!

```typescript
type NewPostPayload = {
  title: string;
  content: string;
  authorId: string;
  postId: string;
};

type NewPostAction = ActionBlueprint<"NEW_POST", NewPostPayload>;

type DeletePostPayload = {
  postId: string;
};

type DeletePostAction = ActionBlueprint<"DELETE_POST", DeletePostPayload>;
```

We'll also make an `AppAction` type that's simply the union of all existing actions in our app:

```typescript
type AppAction = NewPostAction | DeletePostAction; // ...
```

Creating a _new post action_ could be done something like this:

```typescript
const NewPostActionCreator = (
  title: string,
  content: string,
  authorId: string,
  postId: string
): NewPostAction => ({
  type: "NEW_POST",
  payload: {
    title,
    content,
    authorId,
    postId
  }
});
```

And here's how we create _delete post actions_:

```typescript
const DeletePostActionCreator = (postId: PostId): DeletePostAction => ({
  type: "DELETE_POST",
  payload: { postId }
});
```

### Reducer, take one

In a traditional Redux app we'd create a subreducer for `appState.posts`:

```typescript
const postReducer = (postState: PostState, action: AppAction): PostState => {
  switch (action.type) {
    case "NEW_POST":
      return {
        ...postState,
        [action.payload.postId]: {
          title: action.payload.title,
          content: action.payload.content,
          authorId: action.payload.authorId
        }
      };
    case "DELETE_POST": {
      const { [action.payload.postId]: DELETE_ME, ...keepUs } = postState;
      return keepUs;
    }
    default:
      return postState;
  }
};
```

The subreducer for `appState.users` also has to deal with both actions, in order to keep the user postlists up to date:

```typescript
const userReducer = (userState: UserState, action: AppAction): UserState => {
  switch (action.type) {
    case "NEW_POST": {
      const author = userState[action.payload.authorId];
      return {
        ...userState,
        [author.userId]: {
          ...author,
          postIds: author.postIds.concat(action.payload.postId)
        }
      };
    }
    case "DELETE_POST":
      return Object.keys(userState).reduce(
        (mem, userId) => {
          const user = userState[userId];
          return {
            ...mem,
            [userId]: {
              ...user,
              postIds: user.postIds.filter(pid => pid !== action.payload.postId)
            }
          };
        },
        {} as UserState
      );
    default:
      return userState;
  }
};
```

The reducer for our entire app state is merely just putting the subreducers together:

```typescript
type Reducer = (state: AppState, action: AppAction) => AppState;

const reducer: Reducer = (state, action) => ({
  users: userReducer(currentState.users, action),
  posts: postReducer(currentState.posts, action)
});
```

Normally you'd use `Redux.combineReducers`, but that amounts to the same thing.

### Intermission

So far the traditional approach. What's wrong with it?

Well, some would say we've written a fair bit of boilerplate. I'm not sure I fully agree, but definitely it feels verbose.

A worse thing is the shattering of related logic. If you're new to the codebase and want to read up on how post deletion works, you have to look in three places:

- the delete action creator
- the user reducer
- the post reducer

And in reality it is worse - you have to look in the action creator and in ALL subreducers, because you don't really know which ones might have interest in the action!

### Actions 2.0

The pattern we're exploring in this post has a very simple solution to this problem - we _move all related reducing into the action objects_!

An action, apart from the payload, may therefore also include reducers for whatever state domain it wants to affect! Thus we extend our action type accordingly:

```typescript
type ActionBlueprint<T, P> = {
  type: T;
  payload: P;
  reduceUsers?(userState: UserState, payload: P): UserState;
  reducePosts?(postState: PostState, payload: P): PostState;
};
```

Here's what the `DeletePostActionCreator` could look like in this modern take:

```typescript
const DeletePostActionCreator = (postId: string): DeletePostAction => ({
  type: "DELETE_POST",
  payload: { postId },
  reducePosts(postState, payload) {
    const { [payload.postId]: DELETE_ME, ...keepUs } = postState;
    return keepUs;
  },
  reduceUsers(userState, payload) {
    return Object.keys(userState).reduce(
      (mem, userId) => {
        const user = userState[userId];
        return {
          ...mem,
          [userId]: {
            ...user,
            postIds: user.postIds.filter(pid => pid !== payload.postId)
          }
        };
      },
      {} as UserState
    );
  }
});
```

The logic inside the minireducers is what used to live inside individual `case`:s in the old subreducers.

And the `NewPostActionCreator` for completion's sake:

```typescript
const NewPostActionCreator = (
  title: string,
  content: string,
  authorId: string,
  postId: string
): NewPostAction => ({
  type: "NEW_POST",
  payload: {
    title,
    content,
    authorId,
    postId
  },
  reducePosts(postState, payload) {
    return {
      ...postState,
      [payload.postId]: {
        title: payload.title,
        content: payload.content,
        authorId: payload.authorId
      }
    };
  },
  reduceUsers(userState, payload) {
    const author = userState[payload.authorId];
    return {
      ...userState,
      [author.userId]: {
        ...author,
        postIds: author.postIds.concat(payload.postId)
      }
    };
  }
});
```

Note that we don't pass in the full action object to the reducers. Why would we? We are inside a specific creator, so we know what type it is!

### Reducer 2.0

What about the reducers in this brave new world? Well, first off there are no subreducers anymore. The various cases from their switches instead lives in the individual action creators, as you just saw.

The main reducer is simply a matter of applying whatever reducers the action in question happen to have:

```typescript
type Reducer = (
  state: AppState,
  action: ActionBlueprint<string, any>
) => AppState;

const reducer: Reducer = (state, action) => ({
  users: action.reduceUsers
    ? action.reduceUsers(state.users, action.payload)
    : state.users,
  posts: action.reducePosts
    ? action.reducePosts(state.posts, action.payload)
    : state.posts
});
```

### Taking score

So, did we reduce the amount of boilerplate? I think so! The string matching with `action.type` is completely gone, and the Redux-typical switches are nowhere to be seen.

But the big win in my mind is the colocation of related logic. Each action creator file will contain the full truth about how that particular action affects the state.

A third, less obvious win is that we get a clearer separation between actions and thunks. Normally it is very common for the former to be created inline in the latter, but now that the actions carry the reducers it is more obvious that they are their own thing.

### Actions 3.0

But, waitaminue - why are we splitting up the reducing by state domain? Why don't we just do everything in a single reducer for the full state in one fell swoop?

```typescript
type ActionBlueprint<T, P> = {
  type: T;
  payload: P;
  reducer: (state: AppState, payload: P) => AppState;
};
```

The `NewPostActionCreator` can then be streamlined to this...

```typescript
const NewPostActionCreator = (
  title: string,
  content: string,
  authorId: UserId,
  postId: PostId
): NewPostAction => ({
  type: "NEW_POST",
  payload: {
    postId,
    title,
    content,
    authorId
  },
  reducer(state, payload) {
    const author = state.users[payload.authorId];
    return {
      ...state,
      posts: {
        ...state.posts,
        [payload.postId]: {
          title: payload.title,
          content: payload.content,
          authorId: payload.authorId
        }
      },
      users: {
        ...state.users,
        [author.userId]: {
          ...author,
          posts: author.posts.concat(payload.postId)
        }
      }
    };
  }
});
```

...and `DeletePostActionCreator` becomes very short and sweet:

```typescript
const DeletePostActionCreator3 = (postId: PostId): DeletePostAction => ({
  type: "DELETE_POST",
  payload: { postId },
  reducer(state, payload) {
    const { [payload.postId]: postToDelete, ...postsToKeep } = state.posts;
    const author = state.users[postToDelete.authorId];
    return {
      ...state,
      posts: postsToKeep,
      users: {
        ...state.users,
        [postToDelete.authorId]: {
          ...author,
          posts: author.posts.filter(pid => pid !== payload.postId)
        }
      }
    };
  }
});
```

Another cute advantage is that we can now make our action type work for any app by adding the app state type to the generic parameters:

```typescript
type ActionBlueprint<T, P, S> = {
  type: T;
  payload: P;
  reducer: (state: S, payload: P) => S;
};

type NewPostAction = ActionBlueprint<"NEW_POST", NewPostPayload, AppState>;
type DeletePostAction = ActionBlueprint<
  "DELETE_POST",
  DeletePostPayload,
  AppState
>;
```

### Reducer 3.0

Now that each action just has one reducer, the "real" reducer becomes even simpler to define:

```typescript
const reducer: Reducer = (state, action) =>
  action.reducer ? action.reducer(state, action.payload) : state;
```

We still need to check that a reducer exists on the action, since Redux fires its own actions that won't have a built-in reducer.

### Wrapping up

I really like how putting the reducing logic inside the action groups all related code together in one place. And you'll find that employing this pattern means less cruft and boilerplate.

The only downside I can see with the pattern we just explored is that it might surprise developers used to the traditional approach. Other than that it is all wins in my book!

What do _you_ think?
