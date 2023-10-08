# VPASS DELETE PASSWORD DIAGRAM
## Authentication
In the process of viewing passwords as well, users need to be authenticated.

## Delete Password Flow Diagram
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
        conditional{"deletePasswordEndPoint()"}
        master_pass(Master Password Check)
    end

    subgraph Data Layer
        database[(Database)]
    end

    %% Login %%
        actor-- 1 -->providers

    %% Onclick to either view or delete password, check for master password and validity as well
    conditional-->master_pass<-.Check session for master password (5).->database

    %% Get Password request from client %%
        actor-- API call to /api/passId/delete-password endpoint (2) -->conditional

    %% Check if there is a current user
        conditional<-. Get authenticated user (3) .->currentUser

    %% If no user
        conditional-- If not authenticated (4) -->redirect

    %% Check if user exist
        conditional--  Delete Password in DB (6) -->database
```

## Delete Password API Endpoint
```javascript
import { NextResponse } from "next/server";

import { currentProfile } from "@/lib/current-profile";
import { db } from "@/lib/db";

export async function DELETE(req: Request, { params }: { params: { userId, passId } }) {
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

        const password = await.db.password.delete({
            where: {
                id: params.passId,
                profileId: params.userId
            }
        })

        return NextResponse.json({msg: "Password deleted successfully"});
    }
}
```