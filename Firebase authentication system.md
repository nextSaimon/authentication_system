Awesome, Saimon! Let’s set up a **full Next.js 14 App Router project with Firebase Auth** (latest version), including:

* ✅ Signup with email verification
* ✅ Login with secure cookie-based session
* ✅ Middleware for protected routes
* ✅ Auto-refreshing token every hour
* ✅ Logout
* ✅ Clean folder structure

---

## 🚀 Project Structure

```
my-auth-app/
├─ app/
│  ├─ page.js              → Home (optional public page)
│  ├─ login/page.js        → Login page
│  ├─ signup/page.js       → Signup page
│  ├─ dashboard/page.js    → Protected page (authenticated users only)
│  ├─ api/
│  │  ├─ set-token/route.js → Set cookie with token
│  │  └─ logout/route.js    → Clear cookie on logout
├─ components/
│  ├─ AuthSessionSync.js   → Auto-refresh token + sync cookie
├─ lib/
│  └─ firebase/
│     ├─ config.js          → Firebase config + auth
│     └─ admin.js           → Firebase Admin SDK for server verification
├─ middleware.js           → Token check middleware
├─ .env.local              → Environment variables
```

---

## 📦 Install Dependencies

```bash
npx create-next-app my-auth-app --app
cd my-auth-app

npm install firebase firebase-admin
```

---

## 🔐 `lib/firebase/config.js` (Client SDK)

```js
import { initializeApp } from "firebase/app";
import { getAuth } from "firebase/auth";

const firebaseConfig = {
  apiKey: process.env.NEXT_PUBLIC_FIREBASE_API_KEY,
  authDomain: process.env.NEXT_PUBLIC_FIREBASE_AUTH_DOMAIN,
  projectId: process.env.NEXT_PUBLIC_FIREBASE_PROJECT_ID,
  appId: process.env.NEXT_PUBLIC_FIREBASE_APP_ID,
};

const app = initializeApp(firebaseConfig);
const auth = getAuth(app);

export { auth };
```

---

## 🔑 `lib/firebase/admin.js` (Admin SDK)

```js
import { getApps, initializeApp, cert } from "firebase-admin/app";
import { getAuth } from "firebase-admin/auth";

const serviceAccount = JSON.parse(process.env.FIREBASE_ADMIN_SDK);

if (!getApps().length) {
  initializeApp({
    credential: cert(serviceAccount),
  });
}

export const adminAuth = getAuth();
```

In `.env.local`:

```
NEXT_PUBLIC_FIREBASE_API_KEY=...
NEXT_PUBLIC_FIREBASE_AUTH_DOMAIN=...
NEXT_PUBLIC_FIREBASE_PROJECT_ID=...
NEXT_PUBLIC_FIREBASE_APP_ID=...
FIREBASE_ADMIN_SDK={"type":"service_account",...}
```

---

## 🧠 `components/AuthSessionSync.js`

```js
"use client";
import { useEffect } from "react";
import { onIdTokenChanged } from "firebase/auth";
import { auth } from "@/lib/firebase/config";

export default function AuthSessionSync() {
  useEffect(() => {
    const unsubscribe = onIdTokenChanged(auth, async (user) => {
      const token = user ? await user.getIdToken() : null;

      await fetch("/api/set-token", {
        method: "POST",
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify({ token }),
      });
    });

    return () => unsubscribe();
  }, []);

  return null;
}
```

---

## 📄 `app/login/page.js`

\[Click to expand for full component code]

<details><summary>Click to view</summary>

```js
"use client";
import { useState } from "react";
import { useRouter } from "next/navigation";
import { signInWithEmailAndPassword } from "firebase/auth";
import { auth } from "@/lib/firebase/config";

export default function LoginPage() {
  const [email, setEmail] = useState("");
  const [password, setPassword] = useState("");
  const [error, setError] = useState("");
  const router = useRouter();

  const handleLogin = async (e) => {
    e.preventDefault();
    try {
      const userCredential = await signInWithEmailAndPassword(auth, email, password);
      const user = userCredential.user;

      if (!user.emailVerified) {
        setError("Please verify your email before logging in.");
        return;
      }

      const token = await user.getIdToken();
      await fetch("/api/set-token", {
        method: "POST",
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify({ token }),
      });

      router.push("/dashboard");
    } catch (err) {
      setError("Login failed");
    }
  };

  return (
    <form onSubmit={handleLogin} className="max-w-md mx-auto mt-20 space-y-4">
      <input type="email" value={email} onChange={(e) => setEmail(e.target.value)} placeholder="Email" className="p-2 border w-full" />
      <input type="password" value={password} onChange={(e) => setPassword(e.target.value)} placeholder="Password" className="p-2 border w-full" />
      <button className="bg-blue-500 text-white p-2 w-full">Login</button>
      {error && <p className="text-red-500">{error}</p>}
    </form>
  );
}
```

</details>

---

## ✍️ `app/signup/page.js`

```js
"use client";
import { useState } from "react";
import { useRouter } from "next/navigation";
import { auth } from "@/lib/firebase/config";
import { createUserWithEmailAndPassword, sendEmailVerification } from "firebase/auth";

export default function SignupPage() {
  const [email, setEmail] = useState("");
  const [password, setPassword] = useState("");
  const [msg, setMsg] = useState("");
  const router = useRouter();

  const handleSignup = async (e) => {
    e.preventDefault();
    const userCredential = await createUserWithEmailAndPassword(auth, email, password);
    await sendEmailVerification(userCredential.user);
    setMsg("Verification email sent. Please check your inbox.");
  };

  return (
    <form onSubmit={handleSignup} className="max-w-md mx-auto mt-20 space-y-4">
      <input type="email" value={email} onChange={(e) => setEmail(e.target.value)} placeholder="Email" className="p-2 border w-full" />
      <input type="password" value={password} onChange={(e) => setPassword(e.target.value)} placeholder="Password" className="p-2 border w-full" />
      <button className="bg-green-500 text-white p-2 w-full">Signup</button>
      {msg && <p className="text-green-500">{msg}</p>}
    </form>
  );
}
```

---

## 🔐 `app/dashboard/page.js`

```js
"use client";

import { useEffect, useState } from "react";
import { useRouter } from "next/navigation";
import { onAuthStateChanged, signOut } from "firebase/auth";
import { auth } from "@/lib/firebase/config";

export default function Dashboard() {
  const [user, setUser] = useState(null);
  const router = useRouter();

  useEffect(() => {
    const unsubscribe = onAuthStateChanged(auth, (currentUser) => {
      if (currentUser && currentUser.emailVerified) {
        setUser(currentUser);
      } else {
        router.push("/login");
      }
    });

    return () => unsubscribe();
  }, [router]);

  const handleLogout = async () => {
    try {
      await signOut(auth);
      // Clear the token cookie on logout
      await fetch("/api/logout", { method: "POST" });
      router.push("/login");
    } catch (err) {
      console.error("Error signing out:", err);
    }
  };

  if (!user)
    return (
      <div className="text-center mt-10 text-gray-700 font-medium">
        Loading...
      </div>
    );

  return (
    <div className="max-w-2xl mx-auto p-6 bg-white rounded-lg shadow-md mt-10">
      <h1 className="text-2xl font-bold text-gray-800">
        Welcome, {user.email}!
      </h1>
      <p className="mt-4 text-gray-600">This is your dashboard.</p>
      <button
        onClick={handleLogout}
        className="mt-6 w-full p-3 bg-red-500 text-white rounded-md hover:bg-red-600 transition"
      >
        Logout
      </button>
    </div>
  );
}
```

---

## 🍪 `app/api/set-token/route.js`

```js
import { cookies } from "next/headers";

export async function POST(req) {
  const { token } = await req.json();
  if (token) {
    cookies().set("token", token, {
      httpOnly: true,
      secure: process.env.NODE_ENV === "production",
      path: "/",
      maxAge: 60 * 60 * 24,
    });
  } else {
    cookies().delete("token");
  }

  return new Response(JSON.stringify({ success: true }), {
    headers: { "Content-Type": "application/json" },
  });
}
```

---

## 🚪 `app/api/logout/route.js`

```js
import { cookies } from "next/headers";

export async function POST() {
  cookies().delete("token");
  return new Response(JSON.stringify({ success: true }), {
    headers: { "Content-Type": "application/json" },
  });
}
```

---

## 🛡 `middleware.js`

```js
import { NextResponse } from "next/server";
import { adminAuth } from "@/lib/firebase/admin";

export async function middleware(request) {
  const token = request.cookies.get("token")?.value;

  if (!token) return NextResponse.redirect(new URL("/login", request.url));

  try {
    const decoded = await adminAuth.verifyIdToken(token);
    if (!decoded.email_verified) {
      return NextResponse.redirect(new URL("/login", request.url));
    }
    return NextResponse.next();
  } catch (e) {
    return NextResponse.redirect(new URL("/login", request.url));
  }
}

export const config = {
  matcher: ["/dashboard/:path*", "/dashboard"], // Protected routes
};
```

---

## ✅ Next Steps

1. Want `logout` button?
2. Want to add **context** to track user state on client?
3. Deploy to Vercel with environment vars?

Let me know and I’ll help you finish or extend any part of it! Want me to zip and export this whole thing too?
