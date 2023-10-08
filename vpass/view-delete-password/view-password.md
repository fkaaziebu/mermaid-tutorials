# VPASS VIEW PASSWORD DIAGRAM
## Authentication
In the process of viewing passwords as well, users need to be authenticated.

## View Password Flow Diagram
```mermaid
flowchart TB
    subgraph Client Layer
        actor([Client Dashboard])
    end

    subgraph Authentication Layer
        providers{{Clerk}}
        currentUser("currentUser()")
        redirect("redirectToSignUp()")
    end

    subgraph Server Layer
        conditional{"viewPasswordEndPoint()"}
    end

    subgraph Data Layer
        database[(Database)]
    end

    %% Login %%
        actor-- 1 -->providers

    %% Get Password request from client %%
        actor-- API call to /api/passId/view-password endpoint (2) -->conditional

    %% Check if there is a current user
        conditional<-. Get authenticated user (3) .->currentUser

    %% If no user
        conditional-- If not authenticated (4) -->redirect

    %% Check if user exist
        conditional-- Get Password in DB (5) -->database
```

## View Password API Endpoint
```javascript
import { NextResponse } from "next/server";

import { currentProfile } from "@/lib/current-profile";
import { db } from "@/lib/db";

export async function GET(req: Request, { params }: { params: { userId, passId } }) {
    try {
        const profile = await currentProfile();

        if (!profile) {
            return new NextResponse("Unauthorized", { status: 401 });
        }

        if (!params.userId) {
            return new NextResponse("User ID Missing", { status: 400 });
        }

        if (!params.passId) {
            return new NextResponse("Password ID Missing", { status: 400 });
        }

        const password = await.db.password.findUnique({
            where: {
                id: params.passId,
                profileId: params.userId
            },
            include: {
                description,
                password,
                createdAt
            }
        })

        return NextResponse.json(password);
    }
}
```