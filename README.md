## 1. Student Information
* **Name:** MUHAMMAD IRFAN BIN SHAFIE
* **Student ID:** 2024272152
* **Group:** CDCS2703B
* **Lecturer Name:** [Your Lecturer's Name Here]

---

## 2. Project Background
This project is a real-time chat interface built to demonstrate the integration of **Angular 17** with **Supabase** (a backend-as-a-service platform). The application allows users to authenticate via their Google accounts, exchange messages instantly without refreshing the page, and manage their own message history.


## 3. Discussion
3.1 Architectural Choices: SQL vs. NoSQL
For this laboratory submission, I implemented a real-time chat application using Angular 17 and Supabase. While the initial rubric referenced Firebase, I deliberately chose Supabase to explore a relational (SQL-based) approach to real-time data. Supabase provides an open-source alternative to Firebase.
This architecture allowed me to leverage structured data tables (users and chat) rather than unstructured document collections. By using Supabase, I learned how to bridge the gap between traditional SQL database management and modern real-time web sockets. The frontend utilized Angular’s latest features specifically Standalone Components which eliminated the need for NgModules, streamlining the project structure and reducing boilerplate code.


3.2 Technical Implementation
The application relies on a service-based architecture to separate business logic from the user interface.
•	Authentication & Google OAuth 2.0: Instead of simple email/password login, I integrated Google OAuth. This required configuring a project in the Google Cloud Platform (GCP) console to generate a Client ID and Client Secret. These credentials were interconnected with the Supabase Authentication dashboard. In the Angular AuthService, I utilized the supabase.auth.onAuthStateChange listener. This critical method emits events enabling the app to track the user's session state in real-time and update the UI accordingly.
•	Real-time Data Sync: The core feature of the app is instant messaging without page reloads. This was achieved by subscribing to database changes. Unlike a standard REST API that requires a GET request to fetch new data, the Supabase client maintains a socket connection. When a new row is inserted into the chat table, the data is pushed immediately to all connected clients.
•	State Management with Signals: Following the release of Angular 16/17, I moved away from BehaviorSubjects and utilized Angular Signals. Signals provided a more reactive way to handle the message list. When data arrives from Supabase, the signal is updated, and Angular automatically fine-tunes the DOM updates, ensuring high performance.


3.3 Challenges Faced
Developing this application give me several distinct challenges, particularly regarding configuration and security logic.
1. Configuring Google Cloud Platform (GCP): The most tedious part of the setup was correctly configuring the OAuth consent screen and credentials in GCP. A specific hurdle was ensuring the Authorized Redirect URI in Google Console exactly matched the callback URL provided by Supabase. Initially, I encountered mismatch errors where the authentication flow would fail after selecting a Google account. I learned that even a missing trailing slash or incorrect protocol (http vs. https) breaks the handshake between the identity provider and the app.
2. Implementing Row Level Security (RLS) Logic: A key functional requirement was that users should only be able to delete their own messages. While the frontend can hide the "Delete" button visually, this is not secure. The real challenge was understanding that security must be enforced at the database level. I had to ensure that when a delete request is sent, the backend verifies that the user_id of the requester matches the sender_id of the message.
3. Angular 17 Control Flow Migration: As this project used Angular 17, I had to adapt to the new built-in control flow syntax. transitioning from structural directives required a shift in coding style. For example, handling the empty state of the chat (when no messages exist) was cleaner using the @empty block within the @for loop, a feature not present in the older syntax.


3.4 Lessons Learned
Through this lab activity, I gained three significant skills:
1.	Modern Angular Syntax: I am now comfortable using Standalone Components and the new Control Flow syntax, which makes the HTML template much more readable compared to previous versions.
2.	PostgreSQL Triggers: I learned how Supabase uses triggers to automate tasks. For instance, we set up the system so that when a user logs in via Google, their metadata (Avatar URL, Full Name) is automatically captured and inserted into our custom users table. This prevents data duplication and keeps user profiles in sync.
3.	Route Guard Implementation:  I learned how to implement Functional Route Guards. This protects the /chat route, ensuring that if a non-authenticated user tries to access the URL directly, the guard intercepts the navigation and redirects them back to the login page immediately.


3.5 Conclusion
This project successfully demonstrated the creation of a real-time chat application using a modern tech stack. By integrating Angular 17 with Supabase, I achieved a seamless user experience with instant message synchronization and secure Google authentication. Despite the challenges in configuring third-party OAuth providers and adapting to strict SQL security policies, the final application is robust, secure, and performant. This lab has provided a strong foundation in building serverless applications and managing real-time states in a frontend framework.
















<!-- ## Database Table Schema -->
## users table

* id (uuid)
* full_name (text)
* avatar_url (text)

## Creating a users table

```sql
CREATE TABLE public.users (
   id uuid not null references auth.users on delete cascade,
   full_name text NULL,
   avatar_url text NULL,
   primary key (id)
);
```

## Enable Row Level Security

```sql
ALTER TABLE public.users ENABLE ROW LEVEL SECURITY;
```

## Permit Users Access Their Profile

```sql
CREATE POLICY "Permit Users to Access Their Profile"
  ON public.users
  FOR SELECT
  USING ( auth.uid() = id );
```

## Permit Users to Update Their Profile

```sql
CREATE POLICY "Permit Users to Update Their Profile"
  ON public.users
  FOR UPDATE
  USING ( auth.uid() = id );
```

## Supabase Functions

```sql
CREATE
OR REPLACE FUNCTION public.user_profile() RETURNS TRIGGER AS $$ BEGIN INSERT INTO public.users (id, full_name,avatar_url)
VALUES
  (
    NEW.id,
    NEW.raw_user_meta_data ->> 'full_name'::TEXT,
    NEW.raw_user_meta_data ->> 'avatar_url'::TEXT,
  );
RETURN NEW;
END;
$$ LANGUAGE plpgsql SECURITY DEFINER;
```

## Supabase Trigger

```sql
  CREATE TRIGGER
  create_user_trigger
  AFTER INSERT ON auth.users
  FOR EACH ROW
  EXECUTE PROCEDURE
    public.user_profile();
```

## Chat_Messages table (Real Time)

* id (uuid)
* Created At (date)
* text (text)
* editable (boolean)
* sender (uuid)
