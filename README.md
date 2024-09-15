# XRoute

Mobx powered `History` router, with types.

+ [Features](#features)
+ [Usage](#usage)
+ [Troubleshooting](#troubleshooting)
  + [Typescript errors](#typescript-errors)


## Features

- [x] Declarative observable wrapper over the `History` interface
- [x] Type safe `pathname` params
- [x] Type safe `search` params via `qs` serialization
- [x] Type safe `hash` string
- [x] Workarounds for some known issues with History: [Here](https://github.com/ReactTraining/history/issues/811)
- [x] Route inheritance for nesting routes 



## Usage

- [x] Requires >= Mobx@6

> See the [stories for usage examples](./stories/mobx.stories.tsx)

- [docs/routeDefinition.ts](./docs/routeDefinition.ts)

![image](https://github.com/nfour/xroute/assets/2108452/c7b118f8-f180-4e1b-99ca-15b130b71470)

```tsx
import { createBrowserHistory } from 'history'
import { XRoute, XRouter } from 'xroute'

/** A simple route, matches the `/`, the root page */
export const HomeRoute = XRoute('home')
  .Resource('/') // /
  .Type<{
    pathname: {}
    search: {}
  }>()

export const AdminRoute = XRoute('admin')
  .Resource(
    `/admin`, // /admin
  )
  .Type<{
    pathname: {}
    search: { isAdvancedView?: boolean }
  }>()

enum AdminAnalyticsSubSections {
  TopPages = 'topPages',
  TopUsers = 'topUsers',
  RawData = 'rawData',
}

const AdminAnalyticsSubsectionsURI = `:subSection(${AdminAnalyticsSubSections.TopPages}|${AdminAnalyticsSubSections.TopUsers}|${AdminAnalyticsSubSections.RawData})?`

//
// OR:
//    if you dont care about type safety, do this:
//

const AdminAnalyticsSubsectionsURILoose = `:subSection(${Object.values(
  AdminAnalyticsSubSections,
).join('|')})?` as const

export const AdminAnalyticsRoute = AdminRoute.Extend('adminAnalytics')
  .Resource(`/analytics/${AdminAnalyticsSubsectionsURI}`) // /admin/analytics/:subSection(topPages|topUsers|rawData)
  .Type<{
    pathname: { subSection?: AdminAnalyticsSubSections }
    search: {}
  }>()

export const AdminUsersRoute = AdminRoute.Extend('adminUsers')
  .Resource(`/users`) // /admin/users
  .Type<{
    pathname: {}

    // You don't need to use the pathname at all if you want to keep it simple
    // Can even nest objects and arrays.
    search: {
      userId?: string // ends up as ?userId=123
      editor?: {
        line?: string
        activeToolId?: string
        selectedItems?: string[]
      } // ?editor[line]=1&editor[activeToolId]=2&editor[selectedItems]=3&editor[selectedItems]=4
    }
  }>()

export const NotFoundRoute = XRoute('notFound')
  .Resource('/:someGarbage(.*)?') // /:someGarbage(.*)?
  .Type<{
    pathname: {
      /** The pathname that didnt match any route */
      someGarbage?: string
    }
    search: {}
  }>()

export function createRouter() {
  return new XRouter(
    // Order matters, notice the `notFound` route is at the end, to act as a fallback
    [
      AdminAnalyticsRoute, // /admin/analytics/topPages
      AdminUsersRoute, // /admin/users?userId=123&editor[line]=1&editor[activeToolId]=2&editor[selectedItems]=3&editor[selectedItems]=4
      AdminRoute, // /admin
      HomeRoute, // /
      NotFoundRoute, // /asdaskjdkalsdjklasd
    ],
    createBrowserHistory(),
  )
}

```

- [docs/fullExample.tsx](./docs/fullExample.tsx)

```tsx
import { createHashHistory } from 'history'
import { autorun, makeAutoObservable, reaction } from 'mobx'
import { observer } from 'mobx-react-lite'
import * as React from 'react'
import { XRoute, XRouter } from 'xroute'

//
// Define some routes
//

const HomeRoute = XRoute('home')
  .Resource('/') // Optional language param, eg. /en or /
  .Type<{
    pathname: {}
    search: { language?: 'en' | 'da' | 'de' }
  }>()

const UserProfileRoute = HomeRoute.Extend('userProfile')
  .Resource('/user/:userId') // Required language, eg. /da/user/11
  .Type<{
    pathname: { userId: string }
    search: { profileSection: 'profile' | 'preferences' }
  }>()

const router = new XRouter([UserProfileRoute, HomeRoute], createHashHistory())

export type MyXRouter = typeof router

// Log some changes
autorun(() => console.log('Active route:', router.route))

// Navigate to: /?language=en
router.routes.home.push({ pathname: { language: 'en' } })

// Get the pathname, eg. to put inside an <a href="" />
const homeDaUri = router.routes.home.toUri({ pathname: { language: 'da' } }) // "/da"

// Navigates to: /user/11?language=en
router.routes.userProfile.push({
  pathname: { userId: '11' },
  search: { language: 'en' },
})

// Just change the language in the active route.
// This works as long as the parameter is shared between all routes.
// Navigates to: /user/11?language=da
router.route?.push({ pathname: { language: 'da' } })

// Re-use the current language
// Navigates to: /?language=da
router.routes.home.push({
  search: { language: router.route?.search.language },
})

// Provide a route object to route from anywhere:
// Navigate to: /de/user/55
router.push(UserProfileRoute, {
  pathname: { userId: '55' },
  search: { language: 'de' },
})

// Read route properties:

/** This must be read from the `routes.userProfile` for the type to be consistent */
router.routes.userProfile.pathname?.userId // => '55'

/** Because `language` is available on all routes, we can read it from the active route at `router.route` */
router.route?.search?.language

class UserProfilePage {
  constructor(private router: MyXRouter) {
    this.router = router

    makeAutoObservable(this)
  }

  get route() {
    return this.router.routes.userProfile
  }

  get userId() {
    return this.route.pathname?.userId
  }

  get profileSection() {
    return this.route.search?.profileSection
  }

  setUserId(userId: string) {
    // Uses current route params
    this.route.push({ pathname: { userId } })

    //
    // or
    //
    // Explicitly use previous params...
    this.route.pushExact((uri) => ({
      ...uri,
      pathname: { ...uri.pathname, userId },
    }))
  }

  setProfileSection(profileSection: this['profileSection']) {
    this.route.push({ search: { profileSection } }) // sets ?profileSection=""
  }
}

// Play around with user profile:
void (async () => {
  const userProfilePage = new UserProfilePage(router)

  userProfilePage.userId // 55

  userProfilePage.setUserId('200')

  await new Promise((r) => setTimeout(r, 50)) // Give it time to update the URL and come back...

  userProfilePage.userId // 200
})()

const Component = observer(() => {
  const [router] = React.useState(
    () => new XRouter([UserProfileRoute, HomeRoute], createHashHistory()),
  )

  return (
    <>
      <nav>Active Language: {router.route?.search.language}</nav>
      {router.route?.key === 'home' && <div>Home Page!</div>}
      {
        // Or do this:
      }
      {router.routes.userProfile.isActive && (
        <div>
          User Profile! UserID: {router.routes.userProfile.pathname?.userId}
        </div>
      )}
    </>
  )
})

const listenToUserProfileRoute = () => {
  let previousIsActive: boolean

  reaction(
    () => router.routes.userProfile.isActive,
    (isActive) => {
      if (isActive === previousIsActive) return // Ignore same state

      previousIsActive = isActive

      if (isActive) {
        // on enter route
        // ...
      } else {
        // on exit route
        // ...
      }
    },
  )
}

```

## Troubleshooting

### Typescript errors

Are you getting an error like this?

```
The inferred type of "X" cannot be named without a reference to "Y"
```

This can be an issue now that the project uses `zod` for schema generation support.

It can help to update your tsconfig.json with this:

```json
{
  "compilerOptions": {
    "moduleResolution": "NodeNext",
    "module": "NodeNext",
  }
}

```

Or if that doesnt work, add one of these imports anywhere in your project

```ts
import 'xroute/XRouteSchema'
// or if that doesnt work in your older typescript project, use the ugly path:
import 'xroute/x/esm/XRouteSchema'
```