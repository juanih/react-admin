---
layout: default
title: "Authentication"
---

# Authentication

![Login](./img/login.gif)

React-admin lets you secure your admin app with the authentication strategy of your choice. Since there are many possible strategies (Basic Auth, JWT, OAuth, etc.), react-admin delegates authentication logic to your `authProvider`, and provides hooks to execute your authentication code.

## The `authProvider`

By default, react-admin apps don't require authentication. To restrict access to the admin, pass an `authProvider` to the `<Admin>` component.

```jsx
// in src/App.js
import authProvider from './authProvider';

const App = () => (
    <Admin authProvider={authProvider}>
        ...
    </Admin>
);
```

What's an `authProvider`? Just like a `dataProvider`, an `authProvider` is an object that handles authentication logic. It exposes methods that react-admin calls when needed, and that return a Promise. The simplest `authProvider` is:

```js
const authProvider = {
    login: params => Promise.resolve(),
    logout: params => Promise.resolve(),
    checkAuth: params => Promise.resolve(),
    checkError: error => Promise.resolve(),
    getPermissions: params => Promise.resolve(),
    getIdentity: () => Promise.resolve(),
};
```

**Tip**: In react-admin version 2.0, the `authProvider` used to be a function instead of an object. React-admin 3.0 accepts both object and (legacy) function authProviders.

Let's see when react-admin calls the `authProvider`, and how to write one for your own authentication provider. 

## Login Configuration

Once an admin has an `authProvider`, react-admin enables a new page on the `/login` route, which displays a login form asking for a username and password.

![Default Login Form](./img/login-form.png)

Upon submission, this form calls the `authProvider.login({ login, password })` method. It's the ideal place to authenticate the user, and store their credentials.

For instance, to query an authentication route via HTTPS and store the credentials (a token) in local storage, configure `authProvider` as follows:

```js
// in src/authProvider.js
const authProvider = {
    login: ({ username, password }) =>  {
        const request = new Request('https://mydomain.com/authenticate', {
            method: 'POST',
            body: JSON.stringify({ username, password }),
            headers: new Headers({ 'Content-Type': 'application/json' }),
        });
        return fetch(request)
            .then(response => {
                if (response.status < 200 || response.status >= 300) {
                    throw new Error(response.statusText);
                }
                return response.json();
            })
            .then(auth => {
                localStorage.setItem('auth', JSON.stringify(auth));
            });
    },
    // ...
};

export default authProvider;
```

Once the promise resolves, the login form redirects to the previous page, or to the admin index if the user just arrived.

**Tip**: It's a good idea to store credentials in `localStorage`, as in this example, to avoid reconnection when opening a new browser tab. But this makes your application [open to XSS attacks](https://www.redotheweb.com/2015/11/09/api-security.html), so you'd better double down on security, and add an `httpOnly` cookie on the server side, too.

## Sending Credentials to the API

Now the user has logged in, you can use their credentials to communicate with the `dataProvider`. For that, you have to tweak, this time, the `dataProvider` function. As explained in the [Data providers documentation](DataProviders.md#adding-custom-headers), `simpleRestProvider` and `jsonServerProvider` take an `httpClient` as second parameter. That's the place where you can change request headers, cookies, etc.

For instance, to pass the token obtained during login as an `Authorization` header, configure the Data Provider as follows:

```jsx
import { fetchUtils, Admin, Resource } from 'react-admin';
import simpleRestProvider from 'ra-data-simple-rest';

const httpClient = (url, options = {}) => {
    if (!options.headers) {
        options.headers = new Headers({ Accept: 'application/json' });
    }
    const { token } = JSON.parse(localStorage.getItem('auth'));
    options.headers.set('Authorization', `Bearer ${token}`);
    return fetchUtils.fetchJson(url, options);
};
const dataProvider = simpleRestProvider('http://localhost:3000', httpClient);

const App = () => (
    <Admin dataProvider={dataProvider} authProvider={authProvider}>
        ...
    </Admin>
);
```

Now the admin is secured: The user can be authenticated and use their credentials to communicate with a secure API. 

If you have a custom REST client, don't forget to add credentials yourself.

## User Identity 

React-admin displays the current user name and avatar on the top right side of the screen. To enable this feature, implement the `getIdentity` method in the `authProvider`:

```js
// in src/authProvider.js
const authProvider = {
    login: ({ username, password }) => { /* ... */ },
    getIdentity: () => {
        const { id, fullName, avatar } = JSON.parse(localStorage.getItem('auth'));
        return { id, fullName, avatar };
    }
    // ...
};

export default authProvider;
```

React-admin uses the `fullName` and the `avatar` (an image source, or a data-uri) in the App Bar:

![User identity](./img/identity.png)

**Tip**: You can use the `id` field to identify the current user in your code, by calling the `useGetIdentity` hook:

```jsx
import { useGetIdentity, useGetOne } from 'react-admin';

const PostDetail = ({ id }) => {
    const { data: post, loading: postLoading } = useGetOne('posts', id);
    const { identity, loading: identityLoading } = useGetIdentity();
    if (postLoading || identityLoading) return <>Loading...</>;
    if (!post.lockedBy || post.lockedBy === identity.id) {
        // post isn't locked, or is locked by me
        return <PostEdit post={post} />
    } else {
        // post is locked by someone else and cannot be edited
        return <PostShow post={post} />
    }
}
```

## Logout Configuration

As soon as you provide an `authProvider` prop to `<Admin>`, react-admin displays a logout button in the top bar (or in the menu on mobile). When the user clicks on the logout button, this calls the `authProvider.logout()` method, and removes potentially sensitive data from the Redux store. Then the user gets redirected to the login page.

So it's the responsibility of the `authProvider` to clean up the current authentication data. For instance, if the authentication was a token stored in local storage, here the code to remove it:

```js
// in src/authProvider.js
export default {
    login: ({ username, password }) => { /* ... */ },
    getIdentity: () => { /* ... */ },
    logout: () => {
        localStorage.removeItem('auth');
        return Promise.resolve();
    },
    // ...
};
```

![Logout button](./img/logout.gif)

The `authProvider` is also a good place to notify the authentication API that the user credentials are no longer valid after logout.

Note that after logout, react-admin redirects the user to the string returned by `authProvider.logout()` - or to the `/login` url if the method returns nothing. You can customize the redirection url by returning a route string, or `false` to disable redirection after logout. 

## Catching Authentication Errors On The API

If the API requires authentication, and the user credentials are missing in the request or invalid, the API usually answers with an HTTP error code 401 or 403.

Fortunately, each time the API returns an error, react-admin calls the `authProvider.checkError()` method. When `checkError()` returns a rejected promise, react-admin calls the `authProvider.logout()` method.

So it's up to you to decide which HTTP status codes should let the user continue (by returning a resolved promise) or log them out (by returning a rejected promise).

For instance, to log the user out for both 401 and 403 codes:

```js
// in src/authProvider.js
export default {
    login: ({ username, password }) => { /* ... */ },
    getIdentity: () => { /* ... */ },
    logout: () => { /* ... */ },
    checkError: (error) => {
        const status = error.status;
        if (status === 401 || status === 403) {
            localStorage.removeItem('auth');
            return Promise.reject();
        }
        // other error code (404, 500, etc): no need to log out
        return Promise.resolve();
    },
    // ...
};
```

When `authProvider.checkError()` returns a rejected Promise, react-admin redirects to the `/login` page, or to the `error.redirectTo` url. That means you can override the default redirection as follows:

```js
// in src/authProvider.js
export default {
    login: ({ username, password }) => { /* ... */ },
    getIdentity: () => { /* ... */ },
    logout: () => { /* ... */ },
    checkError: (error) => {
        const status = error.status;
        if (status === 401 || status === 403) {
            localStorage.removeItem('auth');
            return Promise.reject({ redirectTo: '/credentials-required' });
        }
        // other error code (404, 500, etc): no need to log out
        return Promise.resolve();
    },
    // ...
};
```

When `authProvider.checkError()` returns a rejected Promise, react-admin displays a notification to the end user, unless the `error.message` is `false`. That means you can disable the notification on error as follows:

```js
// in src/authProvider.js
export default {
    login: ({ username, password }) => { /* ... */ },
    getIdentity: () => { /* ... */ },
    logout: () => { /* ... */ },
    checkError: (error) => {
        const status = error.status;
        if (status === 401 || status === 403) {
            localStorage.removeItem('auth');
            return Promise.reject({ message: false });
        }
        // other error code (404, 500, etc): no need to log out
        return Promise.resolve();
    },
    // ...
};
```

## Checking Credentials During Navigation

Redirecting to the login page whenever a REST response uses a 401 status code is usually not enough, because react-admin keeps data on the client side, and could display stale data while contacting the server - even after the credentials are no longer valid.

Fortunately, each time the user navigates, react-admin calls the `authProvider.checkAuth()` method, so it's the ideal place to validate the credentials.

For instance, to check for the existence of the authentication data in local storage:

```js
// in src/authProvider.js
export default {
    login: ({ username, password }) => { /* ... */ },
    getIdentity: () => { /* ... */ },
    logout: () => { /* ... */ },
    checkError: (error) => { /* ... */ },
    checkAuth: () => localStorage.getItem('auth')
        ? Promise.resolve()
        : Promise.reject(),
    // ...
};
```

If the promise is rejected, react-admin redirects by default to the `/login` page. You can override where to redirect the user in `checkAuth()`, by rejecting an object with a `redirectTo` property:

```js
// in src/authProvider.js
export default {
    login: ({ username, password }) => { /* ... */ },
    getIdentity: () => { /* ... */ },
    logout: () => { /* ... */ },
    checkError: (error) => { /* ... */ },
    checkAuth: () => localStorage.getItem('auth')
        ? Promise.resolve()
        : Promise.reject({ redirectTo: '/no-access' }),
    // ...
}
```

Note that react-admin will call the `authProvider.logout()` method before redirecting. If you specify the `redirectTo` here, it will override the url which may have been returned by the call to `logout()`.

If the promise is rejected, react-admin displays a notification to the end user. You can customize this message by rejecting an error with a `message` property:

```js
// in src/authProvider.js
export default {
    login: ({ username, password }) => { /* ... */ },
    getIdentity: () => { /* ... */ },
    logout: () => { /* ... */ },
    checkError: (error) => { /* ... */ },
    checkAuth: () => localStorage.getItem('auth')
        ? Promise.resolve()
        : Promise.reject({ message: 'login.required' }), // react-admin passes the error message to the translation layer
    // ...
}
```

You can also disable this notification completely by rejecting an error with a `message` with a `false` value:

```js
// in src/authProvider.js
export default {
    login: ({ username, password }) => { /* ... */ },
    getIdentity: () => { /* ... */ },
    logout: () => { /* ... */ },
    checkError: (error) => { /* ... */ },
    checkAuth: () => localStorage.getItem('auth')
        ? Promise.resolve()
        : Promise.reject({ message: false }),
    // ...
}
```

**Tip**: In addition to `login()`, `logout()`, `checkError()`, and `checkAuth()`, react-admin calls the `authProvider.getPermissions()` method to check user permissions. It's useful to enable or disable features on a per user basis. Read the [Authorization Documentation](./Authorization.md) to learn how to implement that type.

## Customizing The Login and Logout Components

Using `authProvider` is enough to implement a full-featured authorization system if the authentication relies on a username and password.

But what if you want to use an email instead of a username? What if you want to use a Single-Sign-On (SSO) with a third-party authentication service? What if you want to use two-factor authentication?

For all these cases, it's up to you to implement your own `LoginPage` component, which will be displayed under the `/login` route instead of the default username/password form, and your own `LogoutButton` component, which will be displayed in the sidebar. Pass both these components to the `<Admin>` component:

```jsx
// in src/App.js
import * as React from "react";
import { Admin } from 'react-admin';

import MyLoginPage from './MyLoginPage';
import MyLogoutButton from './MyLogoutButton';

const App = () => (
    <Admin loginPage={MyLoginPage} logoutButton={MyLogoutButton} authProvider={authProvider}>
    ...
    </Admin>
);
```

Use the `useLogin` and `useLogout` hooks in your custom `LoginPage` and `LogoutButton` components.

```jsx
// in src/MyLoginPage.js
import * as React from 'react';
import { useState } from 'react';
import { useLogin, useNotify, Notification, defaultTheme } from 'react-admin';
import { ThemeProvider } from '@material-ui/styles';
import { createMuiTheme } from '@material-ui/core/styles';

const MyLoginPage = ({ theme }) => {
    const [email, setEmail] = useState('');
    const [password, setPassword] = useState('');
    const login = useLogin();
    const notify = useNotify();
    const submit = e => {
        e.preventDefault();
        login({ email, password }).catch(() =>
            notify('Invalid email or password')
        );
    };

    return (
        <ThemeProvider theme={createMuiTheme(defaultTheme)}>
            <form onSubmit={submit}>
                <input
                    name="email"
                    type="email"
                    value={email}
                    onChange={e => setEmail(e.target.value)}
                />
                <input
                    name="password"
                    type="password"
                    value={password}
                    onChange={e => setPassword(e.target.value)}
                />
            </form>
            <Notification />
        </ThemeProvider>
    );
};

export default MyLoginPage;

// in src/MyLogoutButton.js
import * as React from 'react';
import { forwardRef } from 'react';
import { useLogout } from 'react-admin';
import MenuItem from '@material-ui/core/MenuItem';
import ExitIcon from '@material-ui/icons/PowerSettingsNew';

const MyLogoutButton = forwardRef((props, ref) => {
    const logout = useLogout();
    const handleClick = () => logout();
    return (
        <MenuItem
            onClick={handleClick}
            ref={ref}
        >
            <ExitIcon /> Logout
        </MenuItem>
    );
});

export default MyLogoutButton;
```

**Tip**: By default, react-admin redirects the user to '/login' after they log out. This can be changed by passing the url to redirect to as parameter to the `logout()` function:

```diff
// in src/MyLogoutButton.js
// ...
-   const handleClick = () => logout();
+   const handleClick = () => logout('/custom-login');
```

## `useAuthenticated()` Hook

If you add [custom pages](./Actions.md), or if you [create an admin app from scratch](./CustomApp.md), you may need to secure access to pages manually. That's the purpose of the `useAuthenticated()` hook, which calls the `authProvider.checkAuth()` method on mount, and redirects to login if it returns a rejected Promise.

```jsx
// in src/MyPage.js
import { useAuthenticated } from 'react-admin';

const MyPage = () => {
    useAuthenticated(); // redirects to login if not authenticated
    return (
        <div>
            ...
        </div>
    )
};

export default MyPage;
```

If you call `useAuthenticated()` with a parameter, this parameter is passed to the `authProvider` call as second parameter. that allows you to add authentication logic depending on the context of the call:

```jsx
const MyPage = () => {
    useAuthenticated({ foo: 'bar' }); // calls authProvider.checkAuth({ foo: 'bar' })
    return (
        <div>
            ...
        </div>
    )
};
```

The `useAuthenticated` hook is optimistic: it doesn't block rendering during the `authProvider` call. In the above example, the `MyPage` component renders even before getting the response from the `authProvider`. If the call returns a rejected promise, the hook redirects to the login page, but the user may have seen the content of the `MyPage` component for a brief moment.

## `<Authenticated>` Component

The `<Authenticated>` component uses the `useAuthenticated()` hook, and renders its child component - unless the authentication check fails. Use it as an alternative to the `useAuthenticated()` hook when you can't use a hook, e.g. inside a `Route` `render` function:

```jsx
import { Authenticated } from 'react-admin';

const CustomRoutes = [
    <Route path="/foo" render={() =>
        <Authenticated>
            <Foo />
        </Authenticated>
    } />
];
const App = () => (
    <Admin customRoutes={customRoutes}>
        ...
    </Admin>
);
```

## `useAuthState()` Hook

To avoid rendering a component and force waiting for the `authProvider` response, use the `useAuthState()` hook instead of the `useAuthenticated()` hook. It returns an object with 3 properties:

- `loading`: `true` just after mount, while the `authProvider` is being called. `false` once the `authProvider` has answered.
- `loaded`: the opposite of `loading`.
- `authenticated`: `true` while loading. then `true` or `false` depending on the `authProvider` response.

You can render different content depending on the authenticated status. 

```jsx
import { useAuthState, Loading } from 'react-admin';

const MyPage = () => {
    const { loading, authenticated } = useAuthState();
    if (loading) {
        return <Loading />;
    }
    if (authenticated) {
        return <AuthenticatedContent />;
    } 
    return <AnonymousContent />;
};
```
