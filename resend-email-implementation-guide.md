# Resend Email Implementation Guide

Complete, copy-paste-able implementation guide for a Resend email system in a Next.js (App Router) project. Replace all `Your Website` placeholders with your actual site name.

## Table of Contents

1. [Overview](#overview)
2. [Prerequisites & Installation](#prerequisites--installation)
3. [Resend Account Setup](#resend-account-setup)
4. [Environment Variables](#environment-variables)
5. [Types](#types)
6. [Validation Layer](#validation-layer)
7. [API Routes](#api-routes)
8. [Client-Side Submission Flow](#client-side-submission-flow)
9. [Form Component Wiring](#form-component-wiring)
10. [Newsletter Implementation](#newsletter-implementation)
11. [Request / Response Contract](#request--response-contract)
12. [Flow Diagram](#flow-diagram)
13. [File Reference Map](#file-reference-map)

---

## 1. Overview

Two email flows are implemented with Resend:

| Flow | Purpose | API Route | Sends To |
|------|---------|-----------|----------|
| **Contact Form** | Notify the business owner when a visitor submits the contact form | `/api/resend/contact` | `RESEND_TO_EMAIL` |
| **Newsletter** | Notify the business owner when someone subscribes | `/api/resend/newsletter` | `RESEND_TO_EMAIL` |

Both flows send a **notification email to the site owner** (from `RESEND_FROM_EMAIL` to `RESEND_TO_EMAIL`). They do not send confirmation emails back to the visitor.

The contact flow is a **2-step pipeline**:

1. **Save to database** via `/api/contact` (Prisma) — required
2. **Send notification email** via `/api/resend/contact` (Resend) — non-critical, failure is tolerated

---

## 2. Prerequisites & Installation

Requires Next.js (App Router) with `src/app` directory structure and Node.js.

```bash
npm install resend dotenv
```

Dependencies used in this implementation:

- `resend` `^6.16.0`
- `dotenv` `^17.4.2`

> `dotenv/config` is imported inside the API routes to load `.env` values. If your project loads env vars natively (Next.js does this automatically), the `dotenv` import is optional but harmless.

---

## 3. Resend Account Setup

1. Create an account at [https://resend.com](https://resend.com).
2. Generate an **API Key** in the Resend dashboard (Keys section).
3. **Verify a sender domain** (DNS records) so you can send from `noreply@yourdomain.com` or `contact@yourdomain.com`. Without a verified domain, only the default `onboarding@resend.dev` sender works (and only with your registered account email as recipient).
4. Add the recipient email address (the inbox that should receive the notifications).

---

## 4. Environment Variables

Add to `.env` (and mirror in `.env.example`):

```bash
#Resend
RESEND_API_KEY=
RESEND_FROM_EMAIL=
RESEND_TO_EMAIL=
```

| Variable | Description |
|----------|-------------|
| `RESEND_API_KEY` | API key from the Resend dashboard |
| `RESEND_FROM_EMAIL` | Verified sender address, e.g. `noreply@yourdomain.com` |
| `RESEND_TO_EMAIL` | Recipient inbox for notifications, e.g. `info@yourdomain.com` |

Never commit real values. Use `.env.example` with empty values as the committed template.

---

## 5. Types

`src/types/contact.ts`

```ts
export interface ContactFormData {
  name: string
  email: string
  phone: string
  project: string
  subject: string
  message: string
}

export interface ContactFormErrors {
  name?: string
  email?: string
  phone?: string
  project?: string
  subject?: string
  message?: string
}
```

---

## 6. Validation Layer

Validation happens **twice**:

- **Client-side** (before submit): `validateContactForm` → shows inline errors, uses i18n translation keys.
- **Server-side** (in `/api/contact`): `validateContactSubmission` → hard English error messages, protects the API against bad payloads.

### 6.1 Shared Regex

`src/utils/regex.ts`

```ts
export const EMAIL_REGEX = /^[^\s@]+@[^\s@]+\.[^\s@]+$/
export const BD_PHONE_REGEX = /^(?:\+880|880|0)1[3-9]\d{8}$/
```

- `EMAIL_REGEX` — basic email format check.
- `BD_PHONE_REGEX` — Bangladeshi mobile numbers (`+880`, `880`, or `0` prefix followed by `1[3-9]` and 8 digits).

> This regex is region-specific. Adapt it to your target audience's phone format (e.g. US: `^\+?1?\d{10}$`, EU: `^\+?\d{1,4}[\d\s-]{8,}$`), or use a general `^[+\d][\d\s-]{7,}$` for international numbers.

### 6.2 Client-Side Validation

`src/utils/contact.ts` — `validateContactForm`

```ts
import { EMAIL_REGEX, BD_PHONE_REGEX } from "@/utils/regex"
import type { ContactFormData, ContactFormErrors } from "@/types/contact"

const MAX_NAME = 100
const MAX_EMAIL = 50
const MAX_PHONE = 20
const MAX_PROJECT = 100
const MAX_SUBJECT = 150
const MAX_MESSAGE = 500

export function validateContactForm(data: ContactFormData, t: (key: string) => string): ContactFormErrors {
  const errors: ContactFormErrors = {}

  if (!data.name.trim()) {
    errors.name = t("contact.errors.nameRequired")
  } else if (data.name.trim().length > MAX_NAME) {
    errors.name = `Name must be under ${MAX_NAME} characters`
  }

  if (!data.email.trim()) {
    errors.email = t("contact.errors.emailRequired")
  } else if (data.email.trim().length > MAX_EMAIL) {
    errors.email = `Email must be under ${MAX_EMAIL} characters`
  } else if (!EMAIL_REGEX.test(data.email.trim())) {
    errors.email = t("contact.errors.emailInvalid")
  }

  if (!data.phone.trim()) {
    errors.phone = t("contact.errors.phoneRequired")
  } else if (data.phone.trim().length > MAX_PHONE) {
    errors.phone = `Phone must be under ${MAX_PHONE} characters`
  } else if (!BD_PHONE_REGEX.test(data.phone.trim())) {
    errors.phone = t("contact.errors.phoneInvalid")
  }

  if (!data.project.trim()) {
    errors.project = t("contact.errors.projectRequired")
  } else if (data.project.trim().length > MAX_PROJECT) {
    errors.project = `Project name must be under ${MAX_PROJECT} characters`
  }

  if (!data.subject.trim()) {
    errors.subject = t("contact.errors.subjectRequired")
  } else if (data.subject.trim().length > MAX_SUBJECT) {
    errors.subject = `Subject must be under ${MAX_SUBJECT} characters`
  }

  if (!data.message.trim()) {
    errors.message = t("contact.errors.messageRequired")
  } else if (data.message.trim().length > MAX_MESSAGE) {
    errors.message = `Message must be under ${MAX_MESSAGE} characters`
  }

  return errors
}
```

**Rules applied per field:** required → max length → format regex. Any errors object key (e.g. `errors.name`) is rendered under the field; an empty object means valid.

### 6.3 Server-Side Validation

`src/utils/contact.ts` — `validateContactSubmission`

```ts
interface ContactSubmissionData {
  name: string
  email: string
  phone: string
  project: string
  subject: string
  message: string
}

interface ValidationResult {
  valid: boolean
  errors: string[]
}

export function validateContactSubmission(data: ContactSubmissionData): ValidationResult {
  const errors: string[] = []

  if (!data.name.trim()) {
    errors.push("Name is required")
  } else if (data.name.trim().length > 100) {
    errors.push("Name must be under 100 characters")
  }

  if (!data.email.trim()) {
    errors.push("Email is required")
  } else if (data.email.trim().length > 320) {
    errors.push("Email must be under 320 characters")
  } else if (!EMAIL_REGEX.test(data.email.trim())) {
    errors.push("Please provide a valid email address.")
  }

  if (!data.phone.trim()) {
    errors.push("Phone number is required")
  } else if (data.phone.trim().length > 20) {
    errors.push("Phone must be under 20 characters")
  } else if (!BD_PHONE_REGEX.test(data.phone.trim())) {
    errors.push("Please provide a valid BD phone number.")
  }

  if (!data.project.trim()) {
    errors.push("Project name is required")
  } else if (data.project.trim().length > 100) {
    errors.push("Project must be under 100 characters")
  }

  if (!data.subject.trim()) {
    errors.push("Subject is required")
  } else if (data.subject.trim().length > 150) {
    errors.push("Subject must be under 150 characters")
  }

  if (!data.message.trim()) {
    errors.push("Message is required")
  } else if (data.message.trim().length > 2000) {
    errors.push("Message must be under 2000 characters")
  }

  return { valid: errors.length === 0, errors }
}
```

### 6.4 i18n Error Keys

Client-side validation uses translation keys. Add to your locale files (e.g. `src/locales/en.json`):

```json
"contact": {
  "errors": {
    "nameRequired": "Your name is required",
    "emailRequired": "E-mail is required",
    "emailInvalid": "Please enter a valid e-mail address",
    "phoneRequired": "Phone number is required",
    "phoneInvalid": "Please enter a valid BD phone number",
    "projectRequired": "Project name is required",
    "subjectRequired": "Subject is required",
    "messageRequired": "Message is required"
  }
}
```

If your project is not multilingual, replace `t("...")` calls with the literal strings.

---

## 7. API Routes

### 7.1 Contact Notification

`src/app/api/resend/contact/route.ts`

```ts
import { Resend } from 'resend';
import { NextResponse } from 'next/server';
import 'dotenv/config';

const resend = new Resend(process.env.RESEND_API_KEY || 're_dummy_key');
const fromEmail = process.env.RESEND_FROM_EMAIL;
const toEmail = process.env.RESEND_TO_EMAIL;

export async function POST(request: Request) {
    try {
        const body = await request.json();
        const { name, email, phone, project, subject, message } = body;

        if (!fromEmail || !toEmail) {
            return NextResponse.json({ success: false, message: "Configuration Error" }, { status: 500 });
        }

        const { data, error } = await resend.emails.send({
            from: `Your Website <${fromEmail}>`,
            to: [toEmail],
            subject: `New Contact Request from ${name}`,
            html: `
                <div style="font-family: sans-serif; line-height: 1.6; color: #333;">
                    <h1 style="color: #000;">New Contact Form Submission from Your Website</h1>
                    <p><b>Name:</b> ${name}</p>
                    <p><b>Email:</b> ${email}</p>
                    <p><b>Phone:</b> ${phone || "N/A"}</p>
                    <p><b>Project:</b> ${project || "N/A"}</p>
                    <p><b>Subject:</b> ${subject || "N/A"}</p>
                    <hr style="border: 0; border-top: 1px solid #eee; margin: 20px 0;">
                    <p><b>Message:</b></p>
                    <p style="background: #f9f9f9; padding: 15px; border-radius: 5px;">${message}</p>
                </div>
            `,
        });

        if (error) {
            return NextResponse.json({ success: false, message: error.message }, { status: 400 });
        }

        return NextResponse.json({ success: true, message: "Email Sent Successfully!" }, { status: 200 });
    } catch (error: any) {
        return NextResponse.json({ success: false, message: error.message }, { status: 500 });
    }
}
```

**Key behaviors:**

- Resend client is instantiated once at module level using `RESEND_API_KEY` (with a `re_dummy_key` fallback so the module never crashes without env).
- Config guard: if `RESEND_FROM_EMAIL` / `RESEND_TO_EMAIL` are missing → `500 Configuration Error`.
- Email built with inline-styled HTML (string template).
- Optional fields fall back to `"N/A"` in the email body.
- `resend.emails.send()` returns `{ data, error }`; if `error` exists → `400` with `error.message`.
- Any thrown exception (invalid JSON, bad key) → `500` with the error message.

### 7.2 Newsletter Notification

`src/app/api/resend/newsletter/route.ts`

```ts
import { NextResponse } from 'next/server';
import { Resend } from 'resend';
import 'dotenv/config';

const resend = new Resend(process.env.RESEND_API_KEY || 're_dummy_key');
const fromEmail = process.env.RESEND_FROM_EMAIL;
const toEmail = process.env.RESEND_TO_EMAIL;

export async function POST(request: Request) {
    try {
        const { email } = await request.json();

        if (!fromEmail || !toEmail || !process.env.RESEND_API_KEY) {
            return NextResponse.json({ success: false, message: "Configuration Error" }, { status: 500 });
        }

        if (!email) {
            return NextResponse.json({ success: false, message: "Email is required" }, { status: 400 });
        }

        const { data, error } = await resend.emails.send({
            from: `Your Website <${fromEmail}>`,
            to: [toEmail],
            subject: 'New Newsletter Subscription',
            html: `
                <div style="font-family: sans-serif; line-height: 1.6; color: #333;">
                    <h1 style="color: #000;">New Newsletter Subscriber from Your Website</h1>
                    <p>Someone just subscribed to the newsletter!</p>
                    <p><b>Subscriber Email:</b> ${email}</p>
                </div>
            `,
        });

        if (error) {
            return NextResponse.json({ success: false, message: error.message }, { status: 400 });
        }

        return NextResponse.json({ success: true, message: "Subscribed Successfully!" }, { status: 200 });

    } catch (error: any) {
        return NextResponse.json({ success: false, message: error.message }, { status: 500 });
    }
}
```

**Key behaviors:**

- Same pattern as contact, but only reads `email` from the body.
- Missing `email` → `400 "Email is required"`.
- Config guard also checks `RESEND_API_KEY` itself.

---

## 8. Client-Side Submission Flow

`src/utils/contact.ts` — `submitContactForm`

This is the orchestrator called by the form. It runs **two sequential requests**:

```ts
export async function submitContactForm(
  data: ContactFormData
): Promise<{ success: boolean; message: string }> {
  // 1. Save to DB
  let dbResult: { success: boolean; message: string } = { success: false, message: "Database request failed." }
  try {
    const res = await fetch("/api/contact", {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify(data),
    })
    dbResult = await res.json()
  } catch {
    return { success: false, message: "Network error. Please check your connection and try again." }
  }

  if (!dbResult.success) {
    return dbResult
  }

  // 2. Send email
  let emailResult: { success: boolean } = { success: false }
  try {
    const emailRes = await fetch("/api/resend/contact", {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify(data),
    })
    emailResult = await emailRes.json()
  } catch {
    // Email failed but DB saved — still consider partial success
  }

  // 3. Both must succeed for full success
  if (emailResult.success) {
    return { success: true, message: "Your inquiry has been submitted. We'll be in touch soon." }
  }

  // DB saved but email failed — still success, email is non-critical
  return { success: true, message: "Your inquiry has been submitted. We'll be in touch soon." }
}
```

**Behavior rules:**

1. **DB first, email second.** The DB record is the source of truth.
2. Network failure on the DB step → early return with a failure message (form shows an error toast).
3. If DB fails validation/server error → return that error message verbatim.
4. **Email failure is tolerated.** Even if `/api/resend/contact` fails, the user still sees success — the inquiry is already saved in the DB.

### 8.1 The `/api/contact` DB Route (context)

`src/app/api/contact/route.ts` — this is the Prisma layer that runs *before* the email step:

```ts
import { NextResponse } from "next/server"
import prisma from "@/lib/prisma"
import { validateContactSubmission } from "@/utils/contact"

export async function POST(request: Request) {
  try {
    const body = await request.json()
    const name = String(body?.name ?? "").trim()
    const email = String(body?.email ?? "").trim()
    const phone = String(body?.phone ?? "").trim()
    const project = String(body?.project ?? "").trim()
    const subject = String(body?.subject ?? "").trim()
    const message = String(body?.message ?? "").trim()

    const validation = validateContactSubmission({ name, email, phone, project, subject, message })
    if (!validation.valid) {
      return NextResponse.json(
        { success: false, message: validation.errors[0] },
        { status: 400 }
      )
    }

    const submission = await prisma.contactSubmission.create({
      data: { name, email, phone, project, subject, message },
    })

    const safeData = JSON.parse(
      JSON.stringify(
        { id: submission.id, name: submission.name, email: submission.email, createdAt: submission.createdAt },
        (_key, value) => (typeof value === "bigint" ? Number(value) : value)
      )
    )

    return NextResponse.json(
      { success: true, message: "Consultation request received. We will contact you within 24 business hours.", data: safeData },
      { status: 201 }
    )
  } catch (error) {
    console.error("Contact submission error:", error)
    return NextResponse.json(
      { success: false, message: "Internal server error" },
      { status: 500 }
    )
  }
}
```

> If you don't need DB persistence, `submitContactForm` can skip step 1 and call `/api/resend/contact` directly.

---

## 9. Form Component Wiring

The form is a `"use client"` component. Core state + handlers:

```tsx
"use client"

import { useState, useRef, useEffect } from "react"
import { useLanguage } from "@/context/LanguageContext"
import { toast } from "sonner"
import type { ContactFormData, ContactFormErrors } from "@/types/contact"
import { validateContactForm, submitContactForm } from "@/utils/contact"
```

### 9.1 State

```tsx
const [formData, setFormData] = useState<ContactFormData>({
  name: "",
  email: "",
  phone: "",
  project: "",
  subject: "",
  message: "",
})

const [errors, setErrors] = useState<ContactFormErrors>({})
const [isSubmitting, setIsSubmitting] = useState(false)
const [isSubmitted, setIsSubmitted] = useState(false)
const modalTimeout = useRef<ReturnType<typeof setTimeout>>(undefined)
```

### 9.2 Change Handler

Clears the field's error as soon as the user edits it:

```tsx
const handleChange = (e: React.ChangeEvent<HTMLInputElement | HTMLTextAreaElement>) => {
  const { name, value } = e.target
  setFormData((prev) => ({ ...prev, [name]: value }))
  if (errors[name as keyof ContactFormErrors]) {
    setErrors((prev) => ({ ...prev, [name]: undefined }))
  }
}
```

### 9.3 Submit Handler

```tsx
const handleSubmit = async (e: React.FormEvent) => {
  e.preventDefault()

  const validationErrors = validateContactForm(formData, t)
  if (Object.keys(validationErrors).length > 0) {
    setErrors(validationErrors)
    return
  }

  setErrors({})
  setIsSubmitting(true)

  try {
    const result = await submitContactForm(formData)
    if (!result.success) {
      toast.error(result.message)
      setIsSubmitting(false)
      return
    }
    setErrors({})
    modalTimeout.current = setTimeout(() => {
      setIsSubmitting(false)
      setFormData({
        name: "",
        email: "",
        phone: "",
        project: "",
        subject: "",
        message: "",
      })
      setIsSubmitted(true)
    }, 400)
  } catch {
    toast.error("Network error. Please try again.")
    setIsSubmitting(false)
  }
}

useEffect(() => {
  return () => clearTimeout(modalTimeout.current)
}, [])
```

**Step-by-step flow:**

1. `e.preventDefault()` — stop native submit.
2. Run `validateContactForm`. If any errors → set them, render inline, abort.
3. Clear errors, set `isSubmitting = true` (button shows "Sending..." and is disabled).
4. Await `submitContactForm`.
5. `result.success === false` → `toast.error(result.message)`, re-enable button.
6. Success → wait 400ms (gives the sending state a beat), reset the form, open the success modal.
7. Any uncaught throw → `toast.error("Network error. Please try again.")`.
8. `useEffect` cleanup clears the pending timeout on unmount.

### 9.4 Form Element Notes

- `<form>` has `noValidate` — disables native browser validation so our custom validation fully controls the UX.
- Every input has `required` (accessibility / fallback).
- Inputs show error styling (`outline-red-400`) and an inline `<p className="text-xs text-red-500">{error}</p>` when a field has an error.
- Submit button: `disabled={isSubmitting}` with `disabled:opacity-50 disabled:cursor-not-allowed`, label switches between `contact.sendBtn` and `contact.submitting` ("Sending...").
- Success is shown via a modal (`isSubmitted`) with a checkmark icon and a reset button that calls `closeModal()`.

---

## 10. Newsletter Implementation

### 10.1 Validator

`src/utils/newsletter.ts`

```ts
import { EMAIL_REGEX } from "@/utils/regex"

const MAX_EMAIL_LENGTH = 50

export function validateNewsletterEmail(email: string): string | null {
  const trimmed = email.trim()

  if (!trimmed) {
    return "Email is required"
  }

  if (trimmed.length > MAX_EMAIL_LENGTH) {
    return "Email address is too long"
  }

  if (!EMAIL_REGEX.test(trimmed)) {
    return "Please enter a valid email address"
  }

  return null
}
```

Returns `null` when valid, otherwise a human-readable error string.

### 10.2 Wiring in a Component

The API route + validator exist, but no UI form is currently wired to the newsletter endpoint in this project. To use it, call the endpoint from a client component exactly like the contact flow:

```tsx
async function handleSubscribe(e: React.FormEvent) {
  e.preventDefault()
  const error = validateNewsletterEmail(email)
  if (error) {
    setError(error)
    return
  }
  setSubmitting(true)
  try {
    const res = await fetch("/api/resend/newsletter", {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify({ email }),
    })
    const data = await res.json()
    if (!data.success) {
      toast.error(data.message)
    } else {
      toast.success(data.message)
    }
  } catch {
    toast.error("Network error. Please try again.")
  } finally {
    setSubmitting(false)
  }
}
```

---

## 11. Request / Response Contract

### POST `/api/resend/contact`

**Request body:**

```json
{
  "name": "John Doe",
  "email": "john@example.com",
  "phone": "+8801712345678",
  "project": "Textile Plant",
  "subject": "Site Survey",
  "message": "Need a structural assessment."
}
```

**Responses:**

| Status | Body | When |
|--------|------|------|
| `200` | `{ "success": true, "message": "Email Sent Successfully!" }` | Email sent |
| `400` | `{ "success": false, "message": "<resend error>" }` | Resend returned an error |
| `500` | `{ "success": false, "message": "Configuration Error" }` | Env vars missing |
| `500` | `{ "success": false, "message": "<exception>" }` | Malformed JSON / unknown error |

### POST `/api/resend/newsletter`

**Request body:**

```json
{ "email": "john@example.com" }
```

**Responses:**

| Status | Body | When |
|--------|------|------|
| `200` | `{ "success": true, "message": "Subscribed Successfully!" }` | Email sent |
| `400` | `{ "success": false, "message": "Email is required" }` | Body missing `email` |
| `400` | `{ "success": false, "message": "<resend error>" }` | Resend returned an error |
| `500` | `{ "success": false, "message": "Configuration Error" }` | Env vars / API key missing |
| `500` | `{ "success": false, "message": "<exception>" }` | Malformed JSON / unknown error |

### POST `/api/contact` (DB layer)

| Status | Body | When |
|--------|------|------|
| `201` | `{ "success": true, "message": "...", "data": { id, name, email, createdAt } }` | Saved |
| `400` | `{ "success": false, "message": "<first validation error>" }` | Validation failed |
| `500` | `{ "success": false, "message": "Internal server error" }` | DB/other error |

---

## 12. Flow Diagram

```
Visitor fills ContactForm.tsx
        │
        ▼
handleSubmit ──► validateContactForm() ──invalid──► inline field errors (abort)
        │ valid
        ▼
submitContactForm(formData)
        │
        ├─ 1) fetch POST /api/contact ──► validateContactSubmission()
        │        │ valid                              │ invalid
        │        ▼                                   └► { success:false } ──► toast.error, stop
        │   prisma.contactSubmission.create()
        │        │
        │        ▼
        │   { success:true }
        │        │
        ├─ 2) fetch POST /api/resend/contact ──► resend.emails.send({from, to, subject, html})
        │        │ sent OK                             │ error / network fail
        │        ▼                                    └► TOLERATED (email is non-critical)
        │   { success:true }
        │
        └──► reset form ──► show success modal
```

---

## 13. File Reference Map

| File | Responsibility |
|------|----------------|
| `src/app/api/resend/contact/route.ts` | Contact notification email |
| `src/app/api/resend/newsletter/route.ts` | Newsletter notification email |
| `src/app/api/contact/route.ts` | DB persistence layer (Prisma) |
| `src/utils/contact.ts` | Client + server validation, `submitContactForm` orchestrator |
| `src/utils/newsletter.ts` | Newsletter email validation |
| `src/utils/regex.ts` | Shared `EMAIL_REGEX`, `BD_PHONE_REGEX` |
| `src/types/contact.ts` | `ContactFormData`, `ContactFormErrors` |
| `src/components/contact/ContactForm.tsx` | Contact form UI, state, submit handler, success modal |
| `src/locales/en.json` / `src/locales/bn.json` | `contact.errors.*` translation keys |
| `.env` / `.env.example` | `RESEND_API_KEY`, `RESEND_FROM_EMAIL`, `RESEND_TO_EMAIL` |
| `package.json` | Dependencies: `resend ^6.16.0`, `dotenv ^17.4.2` |
