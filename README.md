# Completing the Migration of Alpha One Labs Learn Platform to Cloudflare Python Workers

## Candidate Info

- Name: Chinmayee Mada  
- GitHub: https://github.com/chinnu8055/  
- Email: madhachinmayee@gmail.com  
- University: Anurag University, Hyderabad  
- Degree: B.Tech, Computer Science and Engineering  
- Time Zone: IST (UTC+5:30)

---

## Bio

I am a Computer Science student with experience in full stack development using React, TypeScript, Supabase, and serverless platforms. I have built projects like a healthcare platform with role-based access, an AI chatbot using LLM APIs, and a mobile app for carbon footprint tracking.

I have been contributing to the Alpha One Labs project with multiple merged pull requests, where I worked on UI improvements, dark mode, leaderboard, and quiz management features. This helped me understand real-world open source workflows, including code reviews and collaboration.

This experience helps me approach the migration with a strong understanding of both the system and practical implementation.

---

## Project Overview

Project Title: Completing the Migration of Alpha One Labs Learn Platform to Cloudflare Python Workers

Goal:  
Complete the migration of the Django-based platform into the Cloudflare Workers based `learn` repository and make it the main system.

Duration:  
Around 480 hours over 12 weeks

---

## Abstract

The Alpha One Labs platform is currently moving from a Django-based system to a Cloudflare Workers based architecture. The Django repository acts as the reference for features, while the `learn` repository is the new system where migration is already started.

The `learn` repo already has some features like authentication, activities, and dashboard. But many features from Django are still missing.

This project focuses on continuing this migration. It is not building from scratch. Existing patterns and models in the `learn` repo will be reused and extended.

By the end, the platform will run completely on Cloudflare Workers as a single system.

---

## Architecture Direction: Modular Repository Design

The platform will follow a modular repository approach.

### Main Repository (learn)
This will contain only core features:
- Authentication and user management  
- Activities, sessions, enrollments  
- Dashboard and participation  

### Separate Repositories
Non-core features will be moved out:
- AI learning system  
- Advanced community features  
- Future integrations  

### Why this approach
- Easier to maintain  
- Features can be developed independently  
- Cleaner codebase  
- Better for contributors  

All parts will still work together as one platform.

---

## Technical Details

### Current Learn Repo

Already available:
- Register and login  
- Activities and sessions  
- Enrollments  
- Basic dashboard  
- Static frontend pages  

Missing:
- Strong encryption  
- Token expiry  
- Update and delete APIs  
- Attendance system  
- Profile features  
- Community features  
- Pagination and optimization  

---

## Phase 1: Security Improvements

- Replace current encryption with AES-GCM using Web Crypto API  
- Add token expiry and refresh  
- Move secrets to environment variables  
- Fix CORS to allow only trusted origins  
- Protect init and seed endpoints  

---

## Phase 2: Backend Structure

Current code is in a single file. This will be split into modules:

- Auth  
- Activities  
- Sessions  
- Enrollments  
- Profiles  
- Community  
- Shared utilities  

This makes the code easier to manage and extend.

---

## Phase 3: Performance Improvements

- Fix N+1 queries using better joins  
- Add pagination using cursor  
- Add search and filters  

---

## Phase 4: Core Features Completion

### Activities and Sessions
- Update and delete APIs  
- Check ownership  

### Enrollments
- Status flow like pending, approved, completed  
- Proper validation  

### Attendance
- Mark attendance  
- View attendance  

### Profiles
- View and update profile  
- Change password  

### Authorization
- Role based access  
- Host access only for allowed users  

---

## Phase 5: Community Features

- Forums  
- Peer connections  
- Study groups  

These features will match what exists in Django.

---

## Phase 6: Frontend

- Continue using HTML, Tailwind, and JavaScript  
- Add UI for all new features  
- Keep it simple and fast  

---

## Phase 7: Elective Feature - AI Learning Path System

This is an additional feature and will be done after migration.

### What it does
- User enters a topic  
- System creates a learning path  
- Content is divided into steps  
- Each step has a quiz  
- User must pass to move forward  

### Extra features
- Support user uploaded notes  
- Use Wikipedia for reliable info  
- Use YouTube for videos  

### System design
- Uses an LLM for content  
- Works with different providers like OpenAI or Groq  
- Stores structured data in database  

### Flow
1. User enters topic  
2. System generates plan  
3. User reviews it  
4. Learns step by step  
5. Completes quizzes  
6. Tracks progress  

---

## Timeline

Community Bonding:
- Understand both codebases  
- Identify missing parts  
- Plan architecture  

Weeks 1 to 3:
- Security fixes  
- Code restructuring  

Weeks 4 to 6:
- Core APIs and features  

Weeks 7 to 8:
- Profiles and authorization  
- Frontend updates  

Midterm:
- Core migration completed  

Weeks 9 to 10:
- Community features  

Weeks 11 to 12:
- AI feature if time allows  
- Testing and documentation  

---

## Benefits

- Completes migration to a better system  
- Faster performance using edge deployment  
- Better security  
- Easier for contributors  
- Ready for future features  

---

## Previous Contributions

- Multiple merged pull requests in Alpha One Labs repository  
- Worked on UI fixes and feature improvements  

Broken Discord and Slack links in the website footer #807,
Duplicate messaging interfaces causing inconsistent behavior #992,
Virtual Lab navigation inconsistency and duplicated Chemistry URL #982,
Fix duplicate locale in Top Contributors link #869,
Update GSOC'25 link to 2026 in website footer #863,
Add delete functionality to options in quiz #843,
Fix dark mode color issues. #818
Improve discoverability of horizontal scrolling on course enrollment table #791


---

## Why This Project

I have already worked on this project, so I understand both the old system and the new one.

This migration is not simple. It needs careful understanding and correct implementation. That is what makes it interesting to me.

---

## Availability

- Around 40 hours per week  
- No other commitments  
- Regular updates  

---

## Post GSoC

- Continue migration of remaining features  
- Improve AI system  
- Keep contributing to the project  

---
