# A06 Redux Notes:

### reactA06.jsx 

In this file, we add an event listener which listens for the page to be loaded. Once it is loaded, we create a store variable by assigning store to calling configureStore();. We also create a root variable by assigning root to calling document.getElementById('root');. This finds the root div inside root.html.erb, the main place where everything in our SPA will live.

To pull everything together, we call ReactDOM.render and pass in the Root component with the store passed into the Root component as props. We also pass in the root variable we just created.

In the end, the code looks like this: 

```javascript
document.addEventListener('DOMContentLoaded', () => {
  const store = configureStore();
  const root = document.getElementById('root');
  ReactDOM.render(<Root store={store} />, root);
});
```



### util/post_api_util.js

These are are $.ajax calls to the database. Open a terminal window with rails routes for reference while creating the functions defined in the specs. These routes will help determine what the method and url for each function is.

Some of these functions take arguments and some do not. fetchPosts takes no arguments as it is just gathering all posts and there is no need to specify an ID in this case. 

fetchPost and deletePost take id as an argument because we must interpolate the id onto the url so that the call knows which post to find or to delete. 

updatePost and createPost both take post as an argument. In addition to taking post as an argument, { post } must be included in the data: field in the updatePost and createPost functions so that the database has the information it needs to make/update the posts. In updatePost, we interpolate ${post.id} as part of the URL. 

Here is the complete code: 

```javascript
export const fetchPosts = () => (
  $.ajax({
    method: 'GET',
    url: 'api/posts'
  })
)

export const fetchPost = (id) => (
  $.ajax({
    method: 'GET',
    url: `api/posts/${id}`
  })
)

export const createPost = (post) => (
  $.ajax({
    method: 'POST',
    url: 'api/posts',
    data: { post }
  })
)

export const updatePost = (post) => (
  $.ajax({
    method: 'PATCH',
    url: `api/posts/${post.id}`,
    data: { post }
  })
)

export const deletePost = (id) => (
  $.ajax({
    method: 'DELETE',
    url: `api/posts/${id}`
  })
)
```

### actions/post_actions.js 

The specs ask for three constants: RECEIVE_ALL_POSTS, RECEIVE_POST, and REMOVE_POST. It is best to define these constants at the top of the file and then immediately write the accompanying regular ol' action creators. Like so: 

```javascript
import * as PostApiUtil from '../util/post_api_util';

export const RECEIVE_ALL_POSTS = 'RECEIVE_ALL_POSTS';
export const RECEIVE_POST = 'RECEIVE_POST';
export const REMOVE_POST = 'REMOVE_POST';

const receiveAllPosts = (posts) => ({
  type: RECEIVE_ALL_POSTS,
  posts
});

const receivePost = (post) => ({
  type: RECEIVE_POST,
  post
});

const removePost = (post) => ({
  type: REMOVE_POST,
  post
});
```

Don't freak when no additional specs beyond the first three run, they need the thunk action creators written in order to pass. But this gets them out of the way. 

Next, the thunk action creators. The names are defined in the specs and are the same as the names of the functions making $.ajax calls that we wrote in post_api_util.js. The accompanying argument that must be passed in (if there is one) is also the same as the one we passed into the $.ajax call. Pull up that file for reference while working on these. 

After we make the $.ajax call by calling PostApiUtil.functionName(arg), we write a promise using .then. After the $.ajax call is complete we take what we received back (either a post or posts) and then dispatch this to the one of the regular action creators we already wrote, making sure to pass in post or posts as an argument to the action creator. 

Errors? Is your shit in camelCase? Are you exporting the thunks? Did you forget an arrow? Arrow in the wrong place? Are you not returning (whether explicitly or implicitly)? Etc. 🤤🙄 If you have one thunk working right use that as your template.

Thunks in totality look like this and go below exporting the constants and above the regular action creators in the file: 

```javascript
export const fetchPosts = () => dispatch => (
  PostApiUtil.fetchPosts().then(posts => dispatch(receiveAllPosts(posts)))
);

export const fetchPost = (id) => dispatch => (
  PostApiUtil.fetchPost(id).then(post => dispatch(receivePost(post)))
);

export const createPost = (post) => dispatch => (
  PostApiUtil.createPost(post).then(post => dispatch(receivePost(post)))
);

export const updatePost = (post) => dispatch => (
  PostApiUtil.updatePost(post).then(post => dispatch(receivePost(post)))
);

export const deletePost = (id) => dispatch => (
  PostApiUtil.deletePost(id).then(post => dispatch(removePost(post)))
);
```

### reducers/posts_reducer.js 

Onto reducers! Start with the PostsReducer. First we want to set the oldState argument being passed in to an empty object as the default state. Then we want to call Object.freeze(oldState) before creating the switch statement. 

The PostsReducer takes in the oldState and an action as arguments. The action references the action creators in the post_actions.js file. We call switch(action.type) and then match each action type to a case within the switch. So we will have case RECEIVE_ALL_POSTS, case RECEIVE_POST, and case REMOVE_POST within the switch statement. 

Within each case, we want to return a totally new state instead of modifying the old state. To accomplish this, we merge an empty object with the old state and then use proprties of the data contained in the action creators(either post or posts) to make the necessary changes to the new state we are returning.

In RECEIVE_ALL_POSTS we want to return a new state with all of the posts. This is simple, we just merge an empty object with action.posts and return it. 

In RECEIVE_POST we want to merge an empty object with the oldState being passed in then either overwrite or add the post passed into the action. We do this by passing in a third argument to merge which is action.post.id as the key and action.post as the value, all wrapped in a JS object. 

In REMOVE_POST we first merge an empty object with the OldState to create a new object that is just a duplicate of the old one. Then we delete the post by calling delete newState[action.post.id];. Finally, don't forget to return the newState. 

Every switch statement needs a default. The default here is just to return oldState. 

Complete code here: 

```javascript
import { RECEIVE_ALL_POSTS,
         RECEIVE_POST,
         REMOVE_POST } from '../actions/post_actions';
import merge from 'lodash/merge';

const PostsReducer = (oldState = {}, action) => {
  Object.freeze(oldState);
  switch(action.type) {
    case RECEIVE_ALL_POSTS:
      return merge({}, action.posts);
    case RECEIVE_POST:
      return merge({}, oldState, {[action.post.id]: action.post})
    case REMOVE_POST:
      let newState = merge({}, oldState);
      delete newState[action.post.id];
      return newState;
    default:
      return oldState;
  }
};
```

export default PostsReducer;


### reducers/root_reducer.js 

The root reducer is quite simple here. Just export and call combineReducers, passing in the PostsReducer under the key posts: 

```javascript
import { combineReducers } from 'redux';
import PostsReducer from './posts_reducer';

export default combineReducers({
  posts: PostsReducer
});
```

### store/store.js

This one is easy as well. Inside of the configureStore function we are just returning calling configureStore() with three arguments: the reducer, the preloaded state, and the applyMiddleware function with thunk passed in. Everything you need in this function is either being imported at the top of the file or passed into the function configureStore (in the case of the preloadedState). 

```javascript
import { createStore, applyMiddleware } from 'redux';
import RootReducer from '../reducers/root_reducer';
import thunk from 'redux-thunk';

const configureStore = (preloadedState = {}) => {
  return createStore(
    RootReducer,
    preloadedState,
    applyMiddleware(thunk)
  );
};

export default configureStore;
```


Yay, we have completed the Redux cycle and 2/3 of the specs are checked off! Onto the Components section to create container components and presentational components. 
