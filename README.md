Social Network Project – 2-Person Work Split
Project Overview
This project is a Facebook-like social network with the following main features:
Followers
Profiles
Posts and comments
Groups
Group events
Private chat
Group chat
Notifications
Authentication using sessions and cookies
SQLite database with migrations
Image/GIF uploads
WebSockets
Dockerized frontend and backend
The recommended approach for two people is to split the project by end-to-end features, not by assigning one person only frontend and the other only backend.
Each person should work across:
```text
Database
   ↓
Go Backend
   ↓
API
   ↓
Frontend
```
This makes the workload more balanced and ensures both team members understand the complete application.
---
Person 1 – Authentication, Profiles, Followers & Posts
Responsibilities
Authentication
Implement:
Registration
Login
Logout
Sessions
Cookies
Authentication middleware
Password hashing with bcrypt
Registration should support:
Email
Password
First name
Last name
Date of birth
Avatar/Image (optional)
Nickname (optional)
About Me (optional)
---
Profiles
Implement:
User profile page
User information
User posts/activity
Followers list
Following list
Public/private profile setting
Profile visibility rules
Profile Rules
Public profile
Any authenticated user can view the profile information.
Private profile
Only followers should be able to view the full profile information.
---
Followers
Implement:
Follow user
Unfollow user
Follow request
Accept follow request
Decline follow request
Automatic following for public profiles
Follow Logic
```text
User A follows User B
        ↓
Is User B public?
   ┌────┴────┐
  Yes        No
   │          │
Follow      Send Request
Directly       │
          Accept / Decline
```
---
Posts
Implement:
Create posts
Display posts
Post feed
Add image/GIF to post
Comments
Add image/GIF to comments
Post Privacy
Support:
Public
Visible to all users.
Almost Private
Visible only to followers.
Private
Visible only to selected followers.
Person 1 is responsible for enforcing these privacy rules on the backend.
---
Image Handling
Support at least:
JPEG
PNG
GIF
Recommended approach:
```text
Uploaded File
     ↓
Validate Type
     ↓
Save to File System
     ↓
Store File Path in SQLite
```
---
Person 1 Database Ownership
Suggested tables:
```text
users
sessions
followers
posts
post_allowed_users
comments
```
Possible responsibility flow:
```text
Register
   ↓
Login
   ↓
Session
   ↓
Profile
   ↓
Follow Users
   ↓
Create Post
   ↓
Feed
   ↓
Comments
```
---
Person 2 – Groups, Events, Chat & Notifications
Groups
Implement:
Create group
Group title
Group description
Browse groups
Group membership
Invite users
Accept invitation
Reject invitation
Request to join group
Accept join request
Reject join request
Group Rules
A group creator can:
Accept join requests
Reject join requests
Existing group members can:
Invite other users
Invited users can:
Accept the invitation
Reject the invitation
---
Group Posts
Implement:
Create group posts
View group posts
Comment on group posts
Only group members should be allowed to access group content.
```text
User
 ↓
Member of Group?
 ┌──────┴──────┐
Yes            No
 │              │
Access        Denied
```
---
Group Events
Implement:
Create event
Event title
Description
Day/time
Event responses
Minimum response options:
Going
Not Going
Only group members should be able to see and respond to group events.
---
Private Chat
Implement WebSocket-based real-time private messaging.
Users should be able to message each other when at least one of them follows the other.
Basic flow:
```text
User A
  │
  │ WebSocket
  ▼
Backend WebSocket Hub
  │
  │ WebSocket
  ▼
User B
```
Messages should support emojis.
---
Group Chat
Each group should have a shared chat room.
Requirements:
Only group members can connect
Messages are sent through WebSockets
All connected group members receive the message
Messages should be stored in SQLite
Example:
```text
         Group WebSocket Room
        /        |         \
       /         |          \
   User A     User B      User C
```
---
Notifications
Implement notifications for at least:
Follow requests
Group invitations
Group join requests
New group events
Notifications must be visible from every page.
Private-message notifications should be treated separately from normal notifications.
Possible flow:
```text
Follow Request ───────┐
Group Invitation ─────┤
Join Request ─────────┼──→ Notifications
Event Created ────────┘
```
---
Person 2 Database Ownership
Suggested tables:
```text
groups
group_members
group_invitations
group_requests
group_posts
group_comments
events
event_responses
private_messages
group_messages
notifications
```
Possible responsibility flow:
```text
Browse Groups
      ↓
Join / Invite
      ↓
Membership
      ↓
Group Posts
      ↓
Events
      ↓
Group Chat

       +

Private Chat
      ↓
WebSocket
      ↓
Notifications
```
---
Shared Responsibilities
Some parts should be designed by both team members before feature development begins.
1. Project Structure
Suggested structure:
```text
social-network/
│
├── backend/
│   ├── handlers/
│   ├── middleware/
│   ├── models/
│   ├── services/
│   ├── websocket/
│   │
│   ├── pkg/
│   │   └── db/
│   │       ├── migrations/
│   │       │   └── sqlite/
│   │       └── sqlite/
│   │
│   └── server.go
│
├── frontend/
│
├── uploads/
│
├── docker-compose.yml
├── Dockerfile.backend
└── Dockerfile.frontend
```
The exact structure can change, but it should remain organized and consistent.
---
Database Design
Before coding features, both team members should create the database design together.
Recommended first step:
```text
Requirements
     ↓
Entities
     ↓
ER Diagram
     ↓
Tables
     ↓
Relationships
     ↓
Migrations
```
Important relationships include:
```text
users
 │
 ├── followers
 ├── posts
 ├── comments
 ├── groups
 ├── messages
 └── notifications
```
Both developers should agree on:
Table names
Column names
Foreign keys
Delete behavior
Primary-key format
Timestamp format
---
API Conventions
Agree on API naming before development.
Recommended pattern:
```text
/api/auth/*
/api/users/*
/api/posts/*
/api/groups/*
/api/events/*
/api/messages/*
/api/notifications/*
/ws
```
Example endpoints:
```text
POST   /api/auth/register
POST   /api/auth/login
POST   /api/auth/logout

GET    /api/users/:id
POST   /api/users/:id/follow
DELETE /api/users/:id/follow

GET    /api/posts
POST   /api/posts
POST   /api/posts/:id/comments

GET    /api/groups
POST   /api/groups
POST   /api/groups/:id/join

GET    /api/notifications

GET    /ws
```
Avoid inconsistent endpoint naming such as:
```text
/getUsers
/create_post
/groups/create
/api/getMessages
```
---
Development Order
Phase 1 – Foundation
Both developers work together on:
Repository setup
Backend structure
Frontend framework setup
SQLite connection
Migration system
Docker setup
API conventions
ER diagram
User model
Authentication middleware
Error-response format
Authentication should be functional early because almost every feature depends on knowing the current user.
---
Phase 2 – Parallel Feature Development
Person 1
```text
Authentication
      ↓
Profiles
      ↓
Followers
      ↓
Posts
      ↓
Comments
      ↓
Post Privacy
```
Person 2
```text
Groups
      ↓
Membership
      ↓
Invitations / Requests
      ↓
Group Posts
      ↓
Events
      ↓
Event Responses
```
Person 2 can use temporary seeded users while Person 1 finishes the complete authentication UI.
---
Phase 3 – Real-Time Features
Person 2 focuses mainly on:
```text
WebSocket Hub
     │
 ┌───┴────────┐
 │            │
Private     Group
Chat         Chat
```
Then:
```text
WebSocket
    ↓
Notifications
```
Meanwhile Person 1 can finish:
Frontend integration
Upload handling
Privacy testing
Feed improvements
Profile UI
Following UI
General integration testing
---
Phase 4 – Integration
Both developers test all cross-feature behavior.
Examples:
```text
Private Profile
      ↓
Follow Request
      ↓
Notification
      ↓
Accept
      ↓
Follower Can See Posts
```
Another example:
```text
Join Group
    ↓
Membership Accepted
    ↓
View Group Posts
    ↓
Receive Group Events
    ↓
Use Group Chat
```
Testing these complete flows is important because many features depend on each other.
---
Migration Strategy
Do not let both developers independently create migrations with conflicting numbers.
Example initial migration allocation:
Person 1
```text
000001_create_users
000002_create_sessions
000003_create_followers
000004_create_posts
000005_create_comments
```
Person 2
```text
000006_create_groups
000007_create_group_members
000008_create_events
000009_create_messages
000010_create_notifications
```
After these migrations exist, always use the next global migration number.
For example:
```text
000011_add_post_privacy
000012_add_group_invites
000013_add_event_responses
```
Rule
Once a migration has been shared or merged:
Do not modify it.
Create a new migration instead.
---
Git Workflow
Recommended branches:
```text
main
│
└── develop
     ├── feature/auth
     ├── feature/profile
     ├── feature/posts
     ├── feature/groups
     ├── feature/events
     ├── feature/chat
     └── feature/notifications
```
Workflow:
```text
Create Feature Branch
        ↓
Implement Feature
        ↓
Test
        ↓
Push
        ↓
Pull Request
        ↓
Partner Reviews
        ↓
Merge into develop
        ↓
Integration Test
        ↓
Merge Stable Version into main
```
Example:
```bash
git checkout develop
git pull
git checkout -b feature/posts
```
After finishing:
```bash
git add .
git commit -m "feat: implement post creation"
git push -u origin feature/posts
```
Then create a pull request into `develop`.
---
Commit Style
Use readable commits.
Examples:
```text
feat: add user registration
feat: implement follow requests
feat: add post privacy
feat: create group events
feat: add websocket private chat

fix: prevent duplicate follow requests
fix: restrict group posts to members

refactor: move database queries into repository package

docs: add API documentation
```
Avoid commits such as:
```text
update
stuff
fix
final
final2
working
```
---
Docker Responsibilities
Both developers should understand the Docker setup.
The project requires:
```text
Frontend Container
        │
        │ HTTP
        ▼
Backend Container
        │
        ▼
SQLite Database
```
You should have separate Docker images for:
Frontend
Backend
A shared `docker-compose.yml` can be used to run the project locally.
---
Recommended Ownership Summary
Feature	Person 1	Person 2
Authentication	✅	Support
Sessions/Cookies	✅	Support
Profiles	✅	
Followers	✅	
Posts	✅	
Comments	✅	
Post Privacy	✅	
Image Uploads	✅	Support
Groups		✅
Group Membership		✅
Group Invitations		✅
Group Posts		✅
Events		✅
Private Chat	Support	✅
Group Chat	Support	✅
WebSockets	Support	✅
Notifications	Support	✅
SQLite Design	✅	✅
Migration Strategy	✅	✅
Docker	✅	✅
Integration Testing	✅	✅
---
Final Work Split
Person 1
Main area: Social identity and content
```text
Authentication
Profiles
Followers
Posts
Comments
Post Privacy
Image Uploads
Feed
```
Person 2
Main area: Communities and real-time communication
```text
Groups
Membership
Invitations
Join Requests
Group Posts
Events
Private Chat
Group Chat
WebSockets
Notifications
```
Both
```text
Architecture
Database Design
Migrations
API Standards
Docker
Git Workflow
Code Reviews
Integration Testing
Final Testing
```
This split keeps the workload reasonably balanced while ensuring both developers gain experience with the database, Go backend, APIs, and frontend.
