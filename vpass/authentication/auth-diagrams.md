# VPASS USER AUTHENTICATION DIAGRAM
## What is authentication?
Authentication is used by a server when the server needs to know exactly who is accessing their information or site. In authentication, ther user or computer has to prove its identity to the server or client.

## VPASS Authentication Diagram
```mermaid
flowchart TB
    subgraph Client Layer
        actor([Client Dashboard])
        return1(returned profile)
    end

    subgraph Authentication Layer
        providers{{Clerk}}
        currentUser("currentUser()")
        redirect("redirectToSignUp()")
    end

    subgraph Server Layer
        conditional{"getUserProfile()"}
        return2(create profile)
    end

    subgraph Data Layer
        database[(Database)]
    end

        %% Login %%
        actor-- 1 -->providers

        %% Auth Provider %%
        providers--ocurrentUser
        providers --oredirect

        %% Get Profile request from client %%
        actor-- Request Profile Info (2) -->conditional

        %% Server Performs operation for request %%

        %% Check if there is a current user
        conditional<-. Get authenticated user (3) .->currentUser
        %% If no user
        conditional-- If not authenticated (4) -->redirect
        %% Check if user exist
        conditional<-. Check if user in DB (5) .->database
        %% Return user profile info
        conditional-- if user_id in DB <br/>return profile (6) -->return1
        %% If user not in DB, create user and return
        conditional-- if not user_id in DB <br/>create user and <br/>return profile (7) -->return2

        return2-..->database
        return2-- 8 -->return1
        return1-->actor
```

## Sign In to VERSITECH PLATFORM
When a user signs-in to the VERSITECH platform, we authomatically have access to their user profile information from the authentication provider.
If using clerk for authentication, we are provided with two api's which will enable us get the currentUser and if that user does not exist we redirect to the sign-up page so they can sign-in.
The two api's can be accessed as follows
```javascript
import { currentUser, redirectToSignIn } from "@clerk/nextjs"
```

## Fetch Profile Information
After user is logged-in using clerk, we can make api calls to get the currentUser, create a profile for that user in our database if user does not exist or return profile information if in database. This kind of operation will be done when user visits the dashboard page.

```javascript
// file name: ./lib/initial-profile.ts

import { currentUser, redirectToSignIn } from "@clerk/nextjs";
import { currentProfile } from "@lib/current-profile";
import { db } from "@lib/db";

export getUserProfile = async () => {
    const user = await currentUser();

    if (!user) {
        return redirectToSignIn();
    }

    const profile = await db.profile.findUnique({
        where: {
            userId: user.id
        }
    })

    if (profile) {
        return profile;
    }

    const newProfile = await db.profile.create({
        data: {
            userId: user.id,
            name: `${user.firstName} ${user.lastName}`,
            imageUrl: user.imageUrl,
            email: user.emailAdresses[0].emailAddress,
        },
    });

    return newProfile;
}
```