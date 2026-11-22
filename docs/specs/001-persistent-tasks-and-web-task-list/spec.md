---
spec_id: 001-persistent-tasks-and-web-task-list
title: Persistent tasks and web task list
status: draft
priority: P1
shape: gitops-app
---

# 001 — Persistent tasks and web task list

## Problem

The Task API keeps tasks only in its process memory today, so every restart or redeploy empties the board, and two running instances would each show different tasks. On top of that, the only way to see or add a task is to call the API by hand; the website shows no tasks at all. People who want to use taskboard27 to note what they have to do lose their tasks and have no screen to work on. Keeping tasks in a Postgres database and giving the website a task list with a create form makes the board usable by people, not only by API clients.

## Stories

As a taskboard user, I want my tasks to still be there after the system is restarted or redeployed, so that I can rely on the board to remember what I have to do.
**Repos:** taskboard27-api, taskboard27-gitops

As a taskboard user, I want to open the website and see all tasks with their due dates, so that I can see what is open without calling an API.
**Repos:** taskboard27-web, taskboard27-api

As a taskboard user, I want to create a task with a title and, if I like, a due date on the website, so that I can write down what I have to do in the browser.
**Repos:** taskboard27-web, taskboard27-api

## Acceptance criteria

- **AC-001.1** Given tasks A, B and C were created through `POST /tasks`, when the API is restarted or redeployed with a new version, then `GET /tasks` returns `200` with exactly A, B and C, oldest first, each with the same `id`, `title` and `due_date` as returned at creation.
- **AC-001.2** Given the API runs with 2 instances, when a task is created through `POST /tasks`, then each of 10 consecutive `GET /tasks` requests that follow includes that task, whichever instance answers.
- **AC-001.3** Given the task database is unreachable, when a client sends `POST /tasks` or `GET /tasks`, then the response is `503` with the message `"task storage unavailable"`, no task is created, and `/ready` answers a non-`200` status; given the database is reachable again, then `GET /tasks` returns every task stored before the outage.
- **AC-001.4** Given 1000 tasks are stored, when a client sends another valid `POST /tasks`, including after a restart of the API, then the response is `409` with the message `"task limit of 1000 reached"` and the list still holds 1000 tasks.
- **AC-001.5** Given tasks are stored, when a person opens the website's start page, then it lists every task oldest first, each showing its title and its due date as `YYYY-MM-DD` or the text `No due date`; with no tasks stored it shows the text `No tasks yet`.
- **AC-001.6** Given the website's start page is open, when a person enters the title `Buy milk`, optionally picks a due date, and presses `Add task`, then the task appears at the end of the list without a full page reload, the form is emptied, and after reloading the page the task is still listed with the same title and due date.
- **AC-001.7** Given the website's start page is open, when a person submits the form with an empty title, a title of only spaces, or a title longer than 200 characters after trimming, then the text `Enter a title of 1 to 200 characters` appears at the title field, the entered values stay in the form, and no task is created.
- **AC-001.8** Given the API is unreachable or answers an error, when a person opens the start page, then the list area shows `Tasks could not be loaded. Try again later.`; when a person submits a valid task and the API answers `409`, then the form shows `Task limit of 1000 reached.`, and for any other failure `The task could not be saved. Try again.`, in both cases keeping the entered values in the form.

## Non-goals

- Editing, completing or deleting tasks, in the API or on the website.
- Users, login, ownership or per-user task lists; anyone who can reach the website or the API sees and creates all tasks.
- Filtering, sorting options, search or pagination of the list.
- Live updates: a task created elsewhere shows on the website after the page is reloaded.
- Carrying over tasks held in the API's memory at the moment the database version is deployed.
- Backups and restore of the task database (a later spec).

## Assumptions

- **A1** The API's task contract stays as built: `POST /tasks` and `GET /tasks`, fields `id`, `title`, `due_date`, the same validation (`422` for invalid title or due date) and ids; only where tasks are kept changes (other option: revise the contract together with persistence; affects AC-001.1, AC-001.4).
- **A2** Tasks held in the API's memory when the database version is first deployed are not carried over; the board starts empty once, as after any restart today (other option: copy them into the database during the switch; affects AC-001.1).
- **A3** The task database is one Postgres database of the system, run in the cluster by the platform's database operator, with credentials the platform generates in the cluster; no paid service is used (other option: a managed cloud database; affects AC-001.1, AC-001.2, AC-001.3).
- **A4** The limit of 1000 tasks stays, now counted over all stored tasks (other option: raise it to 10000 or drop it now that memory is no concern; affects AC-001.4).
- **A5** A database outage answers `503` with `"task storage unavailable"` and makes the API not ready, instead of hanging or answering `500` (other option: `500` with the standard error; affects AC-001.3).
- **A6** The task list and the create form live together on the website's start page, form above the list (other option: a separate `/tasks` page; affects AC-001.5, AC-001.6).
- **A7** The due date is picked with a date input, so the website sends only valid `YYYY-MM-DD` dates or none; the title is checked on the website with the API's rule (trimmed, 1 to 200 characters) before it is sent (other option: rely on the API's `422` only; affects AC-001.6, AC-001.7).
- **A8** Website and API stay open without authentication, following the decision recorded for the API (taskboard27-api spec 001, A8: no authentication until a users/auth spec exists) (other option: require a login before the website can be used; affects AC-001.5, AC-001.6).
- **A9** The website shows the texts `No tasks yet`, `No due date`, `Add task`, `Enter a title of 1 to 200 characters`, `Tasks could not be loaded. Try again later.`, `Task limit of 1000 reached.` and `The task could not be saved. Try again.` in English (other option: other wording; affects AC-001.5 to AC-001.8).
- **A10** The feature runs in every environment the system has (dev, staging, prod); nothing limits it to dev (other option: dev only for now; affects all criteria).

## Open questions

## Changelog

- 2026-11-22 created
- 2026-11-22 priority P1 proposed: P1: normal bug or feature
