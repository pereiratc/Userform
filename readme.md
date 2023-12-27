Go + TypeScript full stack web app, with nextjs, PostgreSQL and Docker
#
go
#
webdev
#
beginners
#
devops
By the end of this, you will understand and create a simple yet complete full stack app using the following:

Next.js 14 (TypeScript)
Tailwind CSS
Go
PostgreSQL
Docker
Docker Compose
There are MANY technologies, but we'll keep the example as basic as possible to make it understandable.

We will proceed with a bottom-up approach, starting with the database and ending with the frontend.

If you prefer a video version

All the code is available for free on GitHub (link in video description).

Architecture
Before we start, here is a simple schema explaining the app's architecture.

Go + TypeScript full stack web app, with next

The frontend is a Next.js app with TypeScript and Tailwind CSS.

The backend is written in Go.

The database is PostgreSQL. We will use Docker to run the database, the backend, and also the frontend (you can also use Vercel). We will use Docker Compose to run the frontend, the backend, and the database together.

Prerequisites
Basic knowledge of what is a frontend, a backend, an API, and a database
Docker installed on your machine
Go ( I will use version 1.20.1)
(optional) Postman or any other tool to make HTTP requests
1. Preparation
Create any folder you want, and then open it with your favorite code editor.
mkdir <YOUR_FOLDER>
cd <YOUR_FOLDER>
code .
Initialize a git repository.
git init
touch .gitignore
Populate the .gitignore file with the following content:
*node_modules
Create a file called compose.yaml in the project's root.
touch compose.yaml
Your projects should look like this:

Go + TypeScript full stack web app, with next

We are ready to create the fullstack app and build it from the bottom up, starting with the database.

After each step, we will test the app's current state to ensure that everything is working as expected.

2. Database
We will use Postgres but not install it on our machine. Instead, we will use Docker to run it in a container. This way, we can easily start and stop the database without installing it on our machine.

Open the file compose.yaml and add the following content:
services:
  db:
    container_name: db
    image: postgres:13
    environment:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres
      POSTGRES_DB: postgres
    ports:
      - 5432:5432
    volumes:
      - pgdata:/var/lib/postgresql/data

volumes:
  pgdata: {}
then type in your terminal
docker compose up -d
This will pull the Postgres image from Docker Hub and start the container. The -d flag means that the container will run in detached mode so we can continue to use the terminal.

Check if the container is running:
docker ps -a
You should see the container running.

Go + TypeScript full stack web app, with next

Step into the db container
docker exec -it db psql -U postgres
Now that you are in the Postgres container, you can type:
\l
\dt
And you should see no relations.

Go + TypeScript full stack web app, with next

You can leave the tab open. We will use it later.

3. Backend
The first step is done. Now, we will create the backend. We will use Go and Mux.

Create a new folder:
mkdir backend
step into the folder:
cd backend
initialize a new Go module by using this command:
go mod init api
The go.mod file should look like this:

Go + TypeScript full stack web app, with next

Install the dependencies:
go get github.com/gorilla/mux github.com/lib/pq
We need just 2 more files for the Go application, including containerization.

You can create these files in different ways. One of them is to create them manually, the other one is to create them with the command line:
touch main.go go.dockerfile
🗒️ main.go file
The main.go file is the main file of the application: it contains all the endpoints and the logic of the app.

Populate the main.go file as follows:
package main

import (
    "database/sql"
    "encoding/json"
    "log"
    "net/http"
    "os"

    "github.com/gorilla/mux"
    _ "github.com/lib/pq"
)

type User struct {
    Id    int    `json:"id"`
    Name  string `json:"name"`
    Email string `json:"email"`
}

// main function
func main() {
    // connect to database
    db, err := sql.Open("postgres", os.Getenv("DATABASE_URL"))
    if err != nil {
        log.Fatal(err)
    }
    defer db.Close()

    // create table if it doesn't exist
    _, err = db.Exec("CREATE TABLE IF NOT EXISTS users (id SERIAL PRIMARY KEY, name TEXT, email TEXT)")
    if err != nil {
        log.Fatal(err)
    }

    // create router
    router := mux.NewRouter()
    router.HandleFunc("/api/go/users", getUsers(db)).Methods("GET")
    router.HandleFunc("/api/go/users", createUser(db)).Methods("POST")
    router.HandleFunc("/api/go/users/{id}", getUser(db)).Methods("GET")
    router.HandleFunc("/api/go/users/{id}", updateUser(db)).Methods("PUT")
    router.HandleFunc("/api/go/users/{id}", deleteUser(db)).Methods("DELETE")

    // wrap the router with CORS and JSON content type middlewares
    enhancedRouter := enableCORS(jsonContentTypeMiddleware(router))
    // start server
    log.Fatal(http.ListenAndServe(":8000", enhancedRouter))
}

func enableCORS(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        // Set CORS headers
        w.Header().Set("Access-Control-Allow-Origin", "*") // Allow any origin
        w.Header().Set("Access-Control-Allow-Methods", "GET, POST, PUT, DELETE, OPTIONS")
        w.Header().Set("Access-Control-Allow-Headers", "Content-Type, Authorization")

        // Check if the request is for CORS preflight
        if r.Method == "OPTIONS" {
            w.WriteHeader(http.StatusOK)
            return
        }

        // Pass down the request to the next middleware (or final handler)
        next.ServeHTTP(w, r)
    })
}

func jsonContentTypeMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        // Set JSON Content-Type
        w.Header().Set("Content-Type", "application/json")
        next.ServeHTTP(w, r)
    })
}

// get all users
func getUsers(db *sql.DB) http.HandlerFunc {
    return func(w http.ResponseWriter, r *http.Request) {
        rows, err := db.Query("SELECT * FROM users")
        if err != nil {
            log.Fatal(err)
        }
        defer rows.Close()

        users := []User{} // array of users
        for rows.Next() {
            var u User
            if err := rows.Scan(&u.Id, &u.Name, &u.Email); err != nil {
                log.Fatal(err)
            }
            users = append(users, u)
        }
        if err := rows.Err(); err != nil {
            log.Fatal(err)
        }

        json.NewEncoder(w).Encode(users)

    }

}

// get user by id
func getUser(db *sql.DB) http.HandlerFunc {
    return func(w http.ResponseWriter, r *http.Request) {
        vars := mux.Vars(r)
        id := vars["id"]

        var u User
        err := db.QueryRow("SELECT * FROM users WHERE id = $1", id).Scan(&u.Id, &u.Name, &u.Email)
        if err != nil {
            w.WriteHeader(http.StatusNotFound)
            return
        }

        json.NewEncoder(w).Encode(u)
    }
}

// create user
func createUser(db *sql.DB) http.HandlerFunc {
    return func(w http.ResponseWriter, r *http.Request) {
        var u User
        json.NewDecoder(r.Body).Decode(&u)

        err := db.QueryRow("INSERT INTO users (name, email) VALUES ($1, $2) RETURNING id", u.Name, u.Email).Scan(&u.Id)
        if err != nil {
            log.Fatal(err)
        }

        json.NewEncoder(w).Encode(u)
    }
}

// update user
func updateUser(db *sql.DB) http.HandlerFunc {
    return func(w http.ResponseWriter, r *http.Request) {
        var u User
        json.NewDecoder(r.Body).Decode(&u)

        vars := mux.Vars(r)
        id := vars["id"]

        // Execute the update query
        _, err := db.Exec("UPDATE users SET name = $1, email = $2 WHERE id = $3", u.Name, u.Email, id)
        if err != nil {
            log.Fatal(err)
        }

        // Retrieve the updated user data from the database
        var updatedUser User
        err = db.QueryRow("SELECT id, name, email FROM users WHERE id = $1", id).Scan(&updatedUser.Id, &updatedUser.Name, &updatedUser.Email)
        if err != nil {
            log.Fatal(err)
        }

        // Send the updated user data in the response
        json.NewEncoder(w).Encode(updatedUser)
    }
}

// delete user
func deleteUser(db *sql.DB) http.HandlerFunc {
    return func(w http.ResponseWriter, r *http.Request) {
        vars := mux.Vars(r)
        id := vars["id"]

        var u User
        err := db.QueryRow("SELECT * FROM users WHERE id = $1", id).Scan(&u.Id, &u.Name, &u.Email)
        if err != nil {
            w.WriteHeader(http.StatusNotFound)
            return
        } else {
            _, err := db.Exec("DELETE FROM users WHERE id = $1", id)
            if err != nil {
                //todo : fix error handling
                w.WriteHeader(http.StatusNotFound)
                return
            }

            json.NewEncoder(w).Encode("User deleted")
        }
    }
}
For an explanation, check https://youtu.be/429-r55KFmM

We are importing:
database/sql as a connector to the Postgres db
encoding/json to work easily with objects in json format
log to log errors
net/http to handle http requests
os to handle environment variables

The struct defined is for an User with an Id (autoincremented by the db), a name and an email.

In the main function do some things:

We connect to the Postgres db setting an evironment variable
when enable CORS and JSON Content-Type middleware
we create a table in the db if it doesn't exist
we use Mux to handle the 5 endpoints
we listen the server on the port 8000
the function jsonContentTypeMiddleware is a middleware function to add a header (application/json) to al the responses. Nice to have the responses formatted properly and ready ot get used from an eventual frontend
then there are 5 controller to Create, Read, Update and Delete users.
🗒️ go.dockerfile file
The go.dockerfile file is the file that will be used to containerize the Go application.

Populate the go.dockerfile file as follows:
# use official Golang image
FROM golang:1.16.3-alpine3.13

# set working directory
WORKDIR /app

# Copy the source code
COPY . .

# Download and install the dependencies
RUN go get -d -v ./...

# Build the Go app
RUN go build -o api .

#EXPOSE the port
EXPOSE 8000

# Run the executable
CMD ["./api"]
For an explanation, check https://youtu.be/429-r55KFmM

🐙 update the compose.yaml file
Update the compose.yaml file in the project's root, adding the goapp service.

Below the updated version:
services:
  goapp:
    container_name: goapp
    image: francescoxx/goapp:1.0.0
    build:
      context: ./backend
      dockerfile: go.dockerfile
    environment:
      DATABASE_URL: 'postgres://postgres:postgres@db:5432/postgres?sslmode=disable'
    ports:
      - '8000:8000'
    depends_on:
      - db
  db:
    container_name: db
    image: postgres:13
    restart: always
    environment:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres
      POSTGRES_DB: postgres
    ports:
      - 5432:5432
    volumes:
      - pgdata:/var/lib/postgresql/data

volumes:
  pgdata:
For an explanation, check: https://youtu.be/429-r55KFmM?si=Lr-B65Hmej-jh5Nb&t=1807

Build the image and run the container
Now, let's build the image and run the container:
docker compose build
docker compose up -d goapp
docker ps -a
Go + TypeScript full stack web app, with next

After this, you can check on the previous tab that the database is still running and that the table has been created.

Go + TypeScript full stack web app, with next

We are now ready to test the backend.

🧪 Test the backend
We are now ready to test the backend.

You can use Postman or any other tool to make HTTP requests.

get all
Yoy can get all the users, but making a GET request to http://localhost:8000/api/go/users

Go + TypeScript full stack web app, with next

create a new user
You can create a new user, but making a POST request to http://localhost:8000/api/go/users

Go + TypeScript full stack web app, with next

update a user
You can update a user, but making a PUT request to http://localhost:8000/api/go/users/3

Go + TypeScript full stack web app, with next

We can check the content of the database with the following command:
docker exec -it db psql -U postgres
\dt
select * from users;
We can also check it on the browser at http://localhost:8000/api/go/users

4. Frontend
Now that we have the backend up and running, we can proceed with the frontend.

We will use Next.js 14 with TypeScript and Tailwind.

From the root folder of the project,
cd ..
And from the root folder of the project, run this command:
npx create-next-app@latest --no-git
We use the --no-git flag because we already initialized a git repository at the project's root.

As options:

What is your project named? frontend
TypeScript? Yes
EsLint? Yes
Tailwind CSS? Yes
Use the default directory structure? Yes
App Router? No (not needed for this project)
Customize the default import alias? No
This should create a new Next.js project in about one minute.

Go + TypeScript full stack web app, with next

Step into the frontend folder:
cd frontend
Install Axios, we will use it to make HTTP requests (be sure to be in the frontend folder):
npm i axios
Before we proceed, try to run the project:
npm run dev
And open your browser at http://localhost:3000. You should see the default Next.js page.

Go + TypeScript full stack web app, with next

🖋️ Modify the styles/global.css file
In the src/frontend/src/styles/globals.css file, replace the content with this one (to avoid some problems with Tailwind):
@tailwind base;
@tailwind components;
@tailwind utilities;

:root {
  --foreground-rgb: 0, 0, 0;
  --background-start-rgb: 214, 219, 220;
  --background-end-rgb: 255, 255, 255;
}

body {
  color: rgb(var(--foreground-rgb));
  background: linear-gradient(to bottom, transparent, rgb(var(--background-end-rgb))) rgb(var(--background-start-rgb));
}
Create new components
In the /frontend/src folder, create a new folder called components and inside it create a new file called CardComponent.tsx and add the following content:
import React from 'react';

interface Card {
  id: number;
  name: string;
  email: string;
}

const CardComponent: React.FC<{ card: Card }> = ({ card }) => {
  return (
    <div className="bg-white shadow-lg rounded-lg p-2 mb-2 hover:bg-gray-100">
      <div className="text-sm text-gray-600">Id: {card.id}</div>
      <div className="text-lg font-semibold text-gray-800">{card.name}</div>
      <div className="text-md text-gray-700">{card.email}</div>
    </div>
  );
};

export default CardComponent;
Create a UserInterface component
In the /frontend/src/components folder, create a file called UserInterface.tsx and add the following content:
import React, { useState, useEffect } from 'react';
import axios from 'axios';
import CardComponent from './CardComponent';

interface User {
  id: number;
  name: string;
  email: string;
}

interface UserInterfaceProps {
  backendName: string;
}

const UserInterface: React.FC<UserInterfaceProps> = ({ backendName }) => {
  const apiUrl = process.env.NEXT_PUBLIC_API_URL || 'http://localhost:8000';
  const [users, setUsers] = useState<User[]>([]);
  const [newUser, setNewUser] = useState({ name: '', email: '' });
  const [updateUser, setUpdateUser] = useState({ id: '', name: '', email: '' });

  // Define styles based on the backend name
  const backgroundColors: { [key: string]: string } = {
    go: 'bg-cyan-500',
  };

  const buttonColors: { [key: string]: string } = {
    go: 'bg-cyan-700 hover:bg-blue-600',
  };

  const bgColor = backgroundColors[backendName as keyof typeof backgroundColors] || 'bg-gray-200';
  const btnColor = buttonColors[backendName as keyof typeof buttonColors] || 'bg-gray-500 hover:bg-gray-600';

  // Fetch users
  useEffect(() => {
    const fetchData = async () => {
      try {
        const response = await axios.get(`${apiUrl}/api/${backendName}/users`);
        setUsers(response.data.reverse());
      } catch (error) {
        console.error('Error fetching data:', error);
      }
    };

    fetchData();
  }, [backendName, apiUrl]);

  // Create a user
  const createUser = async (e: React.FormEvent<HTMLFormElement>) => {
    e.preventDefault();
    try {
      const response = await axios.post(`${apiUrl}/api/${backendName}/users`, newUser);
      setUsers([response.data, ...users]);
      setNewUser({ name: '', email: '' });
    } catch (error) {
      console.error('Error creating user:', error);
    }
  };

  // Update a user
  const handleUpdateUser = async (e: React.FormEvent<HTMLFormElement>) => {
    e.preventDefault();
    try {
      await axios.put(`${apiUrl}/api/${backendName}/users/${updateUser.id}`, { name: updateUser.name, email: updateUser.email });
      setUpdateUser({ id: '', name: '', email: '' });
      setUsers(
        users.map((user) => {
          if (user.id === parseInt(updateUser.id)) {
            return { ...user, name: updateUser.name, email: updateUser.email };
          }
          return user;
        })
      );
    } catch (error) {
      console.error('Error updating user:', error);
    }
  };

  // Delete a user
  const deleteUser = async (userId: number) => {
    try {
      await axios.delete(`${apiUrl}/api/${backendName}/users/${userId}`);
      setUsers(users.filter((user) => user.id !== userId));
    } catch (error) {
      console.error('Error deleting user:', error);
    }
  };

  return (
    <div className={`user-interface ${bgColor} ${backendName} w-full max-w-md p-4 my-4 rounded shadow`}>
      <img src={`/${backendName}logo.svg`} alt={`${backendName} Logo`} className="w-20 h-20 mb-6 mx-auto" />
      <h2 className="text-xl font-bold text-center text-white mb-6">{`${backendName.charAt(0).toUpperCase() + backendName.slice(1)} Backend`}</h2>

      {/* Form to add new user */}
      <form onSubmit={createUser} className="mb-6 p-4 bg-blue-100 rounded shadow">
        <input
          placeholder="Name"
          value={newUser.name}
          onChange={(e) => setNewUser({ ...newUser, name: e.target.value })}
          className="mb-2 w-full p-2 border border-gray-300 rounded"
        />

        <input
          placeholder="Email"
          value={newUser.email}
          onChange={(e) => setNewUser({ ...newUser, email: e.target.value })}
          className="mb-2 w-full p-2 border border-gray-300 rounded"
        />
        <button type="submit" className="w-full p-2 text-white bg-blue-500 rounded hover:bg-blue-600">
          Add User
        </button>
      </form>

      {/* Form to update user */}
      <form onSubmit={handleUpdateUser} className="mb-6 p-4 bg-blue-100 rounded shadow">
        <input
          placeholder="User Id"
          value={updateUser.id}
          onChange={(e) => setUpdateUser({ ...updateUser, id: e.target.value })}
          className="mb-2 w-full p-2 border border-gray-300 rounded"
        />
        <input
          placeholder="New Name"
          value={updateUser.name}
          onChange={(e) => setUpdateUser({ ...updateUser, name: e.target.value })}
          className="mb-2 w-full p-2 border border-gray-300 rounded"
        />
        <input
          placeholder="New Email"
          value={updateUser.email}
          onChange={(e) => setUpdateUser({ ...updateUser, email: e.target.value })}
          className="mb-2 w-full p-2 border border-gray-300 rounded"
        />
        <button type="submit" className="w-full p-2 text-white bg-green-500 rounded hover:bg-green-600">
          Update User
        </button>
      </form>

      {/* Display users */}
      <div className="space-y-4">
        {users.map((user) => (
          <div key={user.id} className="flex items-center justify-between bg-white p-4 rounded-lg shadow">
            <CardComponent card={user} />
            <button onClick={() => deleteUser(user.id)} className={`${btnColor} text-white py-2 px-4 rounded`}>
              Delete User
            </button>
          </div>
        ))}
      </div>
    </div>
  );
};

export default UserInterface;
For an explanation, check: https://youtu.be/429-r55KFmM

Modify the index.tsx file
Opne the index.tsx file and replace the content with the following:
import React from 'react';
import UserInterface from '../components/UserInterface';

const Home: React.FC = () => {
  return (
    <main className="flex flex-wrap justify-center items-start min-h-screen bg-gray-100">
      <div className="m-4">
        <UserInterface backendName="go" />
      </div>
    </main>
  );
};

export default Home;
For the explanation, check: https://youtu.be/429-r55KFmM

Add the Go logo
In the /frontend/public folder, add the gologo.svg file.

Refresh the page and you should see the Go logo.

Go + TypeScript full stack web app, with next

🧪 Test the frontend
We are now ready to test the frontend.

You can use the UI to insert, update, and delete users.

You can create a user directly from the UI

Go + TypeScript full stack web app, with next

You can also update a user

Go + TypeScript full stack web app, with next

And finally, you can delete a user, just by clicking on the "Delete User' button

Go + TypeScript full stack web app, with next

You can check the content of the database with the following command:
docker exec -it db psql -U postgres
select * from users;
Go + TypeScript full stack web app, with next

Dockerize the frontend
Deploy a Next.js app with Docker.

Change the next.config.js file in the frontend folder, replacing it with the following content:
/** @type {import('next').NextConfig} */
const nextConfig = {
  output: 'standalone',
};

module.exports = nextConfig;
Create a file called .dockerignore in the frontend folder and add the following content:
**/node_modules
To dockerize the Next.js application, we will use the official Dockerfile provided by Vercel:

You can find it here: https://github.com/vercel/next.js/blob/canary/examples/with-docker/Dockerfile

Create a file called next.dockerfile in the frontend folder and add the following content (it's directly from the vercel official docker example)
FROM node:18-alpine AS base

# Install dependencies only when needed
FROM base AS deps
# Check https://github.com/nodejs/docker-node/tree/b4117f9333da4138b03a546ec926ef50a31506c3#nodealpine to understand why libc6-compat might be needed.
RUN apk add --no-cache libc6-compat
WORKDIR /app

# Install dependencies based on the preferred package manager
COPY package.json yarn.lock* package-lock.json* pnpm-lock.yaml* ./
RUN \
  if [ -f yarn.lock ]; then yarn --frozen-lockfile; \
  elif [ -f package-lock.json ]; then npm ci; \
  elif [ -f pnpm-lock.yaml ]; then yarn global add pnpm && pnpm i --frozen-lockfile; \
  else echo "Lockfile not found." && exit 1; \
  fi


# Rebuild the source code only when needed
FROM base AS builder
WORKDIR /app
COPY --from=deps /app/node_modules ./node_modules
COPY . .

# Next.js collects completely anonymous telemetry data about general usage.
# Learn more here: https://nextjs.org/telemetry
# Uncomment the following line in case you want to disable telemetry during the build.
# ENV NEXT_TELEMETRY_DISABLED 1

RUN yarn build && ls -l /app/.next


# If using npm comment out above and use below instead
# RUN npm run build

# Production image, copy all the files and run next
FROM base AS runner
WORKDIR /app

ENV NODE_ENV production
# Uncomment the following line in case you want to disable telemetry during runtime.
# ENV NEXT_TELEMETRY_DISABLED 1

RUN addgroup --system --gid 1001 nodejs
RUN adduser --system --uid 1001 nextjs

COPY --from=builder /app/public ./public

# Set the correct permission for prerender cache
RUN mkdir .next
RUN chown nextjs:nodejs .next

# Automatically leverage output traces to reduce image size
# https://nextjs.org/docs/advanced-features/output-file-tracing
COPY --from=builder --chown=nextjs:nodejs /app/.next/standalone ./
COPY --from=builder --chown=nextjs:nodejs /app/.next/static ./.next/static

USER nextjs

EXPOSE 3000

ENV PORT 3000
# set hostname to localhost
ENV HOSTNAME "0.0.0.0"

# server.js is created by next build from the standalone output
# https://nextjs.org/docs/pages/api-reference/next-config-js/output
CMD ["node", "server.js"]
Now, let's update the compose.yaml file in the project's root, adding the nextapp service.

Below the updated version:
services:
  nextapp:
    container_name: nextapp
    image: nextapp:1.0.0
    build:
      context: ./frontend
      dockerfile: next.dockerfile
    ports:
      - "3000:3000"
    environment:
      - NEXT_PUBLIC_API_URL=http://localhost:8000
    depends_on:
      - goapp
  goapp:
    container_name: goapp
    image: francescoxx/goapp:1.0.0
    build:
      context: ./backend
      dockerfile: go.dockerfile
    environment:
      DATABASE_URL: "postgres://postgres:postgres@db:5432/postgres?sslmode=disable"
    ports:
      - "8000:8000"
    depends_on:
      - db
  db:
    container_name: db
    image: postgres:13
    restart: always
    environment:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres
      POSTGRES_DB: postgres
    ports:
      - 5432:5432
    volumes:
      - pgdata:/var/lib/postgresql/data

volumes:
  pgdata:

And now, let's build the image and run the container:
docker compose build
docker compose up -d nextapp
You can check if the 3 containers are running:
docker ps -a
Go + TypeScript full stack web app, with next

If you have the 3 services running, should be good to go.

Before we wrap up, let's make a final test using the UI.

🧪 Test the frontend
As a final test, we can check if the frontend is working.

To create a new user, add a name and email

Go + TypeScript full stack web app, with next

We can check the list of users from the UI or directly from the database:
docker exec -it db psql -U postgres
\dt
select * from users;
Go + TypeScript full stack web app, with next

📝 Recap
We build a simple yet complete full-stack web app with Rust API, Next.js 14, Serde, Postgres, Docker, docker Compose.

We used Rust to build the backend API, Next.js 14 to build the frontend, Serde to serialize and deserialize the data, Postgres as the database, Docker to containerize the app, and docker Compose to run the app.