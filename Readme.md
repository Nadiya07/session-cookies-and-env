Express Authentication App
Overview
This is an Express.js application that provides user registration, login, and protected routes using Passport.js for authentication and bcrypt for password hashing. The application connects to a PostgreSQL database to store user information.

Features
User Registration
User Login
Password Hashing with bcrypt
Authentication with Passport.js
Session Management with express-session
Protected Routes
Prerequisites
Node.js
PostgreSQL
Installation
Clone the repository:

bash
Copy code
git clone <repository-url>
cd <repository-directory>
Install dependencies:

bash
Copy code
npm install
Set up environment variables:
Create a .env file in the root of the project and add the following environment variables:

env
Copy code
PG_USER=<your_postgres_user>
PG_HOST=<your_postgres_host>
PG_DATABASE=<your_postgres_database>
PG_PASSWORD=<your_postgres_password>
PG_PORT=<your_postgres_port>
SESSION_SECRET=<your_session_secret>
Set up the database:
Create a PostgreSQL database and a users table with the following SQL:

sql
Copy code
CREATE TABLE users (
  id SERIAL PRIMARY KEY,
  email VARCHAR(255) UNIQUE NOT NULL,
  password VARCHAR(255) NOT NULL
);
Run the application:

bash
Copy code
npm start
Usage
Endpoints
GET /: Render the home page.
GET /login: Render the login page.
GET /register: Render the registration page.
GET /logout: Logout the user and redirect to the home page.
GET /secrets: Render the secrets page if the user is authenticated.
POST /login: Authenticate the user and redirect to the secrets page.
POST /register: Register a new user, hash the password, save the user to the database, and redirect to the secrets page.
Protected Routes
/secrets: This route is protected and only accessible to authenticated users. If the user is not authenticated, they will be redirected to the login page.
Middleware
bodyParser: Parses incoming request bodies.
express.static: Serves static files from the public directory.
express-session: Manages user sessions.
passport: Initializes Passport and manages authentication sessions.
Code Explanation
Express Setup
javascript
Copy code
import express from "express";
import bodyParser from "body-parser";
import pg from "pg";
import bcrypt from "bcrypt";
import passport from "passport";
import { Strategy } from "passport-local";
import session from "express-session";
import env from "dotenv";

const app = express();
const port = 3000;
const saltRounds = 10;
env.config();

app.use(session({
  secret: process.env.SESSION_SECRET,
  resave: false,
  saveUninitialized: true,
}));

app.use(bodyParser.urlencoded({ extended: true }));
app.use(express.static("public"));

app.use(passport.initialize());
app.use(passport.session());
Database Connection
javascript
Copy code
const db = new pg.Client({
  user: process.env.PG_USER,
  host: process.env.PG_HOST,
  database: process.env.PG_DATABASE,
  password: process.env.PG_PASSWORD,
  port: process.env.PG_PORT,
});
db.connect();
Routes
javascript
Copy code
app.get("/", (req, res) => {
  res.render("home.ejs");
});

app.get("/login", (req, res) => {
  res.render("login.ejs");
});

app.get("/register", (req, res) => {
  res.render("register.ejs");
});

app.get("/logout", (req, res) => {
  req.logout(function(err) {
    if (err) {
      return next(err);
    }
    res.redirect("/");
  });
});

app.get("/secrets", (req, res) => {
  if (req.isAuthenticated()) {
    res.render("secrets.ejs");
  } else {
    res.redirect("/login");
  }
});

app.post("/login",
  passport.authenticate("local", {
    successRedirect: "/secrets",
    failureRedirect: "/login",
  })
);

app.post("/register", async (req, res) => {
  const email = req.body.username;
  const password = req.body.password;

  try {
    const checkResult = await db.query("SELECT * FROM users WHERE email = $1", [email]);

    if (checkResult.rows.length > 0) {
      res.redirect("/login");
    } else {
      bcrypt.hash(password, saltRounds, async (err, hash) => {
        if (err) {
          console.error("Error hashing password:", err);
        } else {
          const result = await db.query(
            "INSERT INTO users (email, password) VALUES ($1, $2) RETURNING *",
            [email, hash]
          );
          const user = result.rows[0];
          req.login(user, (err) => {
            res.redirect("/secrets");
          });
        }
      });
    }
  } catch (err) {
    console.log(err);
  }
});
Passport Configuration
javascript
Copy code
passport.use(
  new Strategy(async function verify(username, password, cb) {
    try {
      const result = await db.query("SELECT * FROM users WHERE email = $1", [username]);
      if (result.rows.length > 0) {
        const user = result.rows[0];
        const storedHashedPassword = user.password;
        bcrypt.compare(password, storedHashedPassword, (err, valid) => {
          if (err) {
            return cb(err);
          } else {
            if (valid) {
              return cb(null, user);
            } else {
              return cb(null, false);
            }
          }
        });
      } else {
        res.send("User not found");
      }
    } catch (err) {
      console.log(err);
    }
  })
);

passport.serializeUser((user, cb) => {
  cb(null, user);
});

passport.deserializeUser((user, cb) => {
  cb(null, user);
});
Starting the Server
javascript
Copy code
app.listen(port, () => {
  console.log(`Server running on port ${port}`);
});
License
This project is licensed under the MIT License.








