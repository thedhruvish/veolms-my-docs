# VEO-LMS Plugin System Design Draft - 1

## Overview

The LMS will use a **Plugin System** that allows features to be installed, updated, enabled, disabled, or removed without modifying the core application.

The primary goals are:

* Keep the core application lightweight.
* Allow features to be installed only when needed.
* Support third-party extensions.
* Reduce maintenance by keeping each feature isolated.
* Provide a one-command installation experience similar to **shadcn/ui**.

---

# Design Principles

Every feature should be treated as a plugin.

Examples:

* Course Management
* Quiz
* Assignment
* Certificate
* Payment
* Notifications
* Live Classes
* Attendance
* Analytics

The core application should never contain business logic for these features.

Instead, the core provides:

* Plugin Manager
* Event System
* Service Registry
* Configuration Manager
* Database Migration System
* CLI Installer

Everything else is delivered through plugins.

---

# Plugin Lifecycle

Every plugin follows the same lifecycle.

1. Install
2. Configure
3. Register
4. Enable
5. Update
6. Disable
7. Remove

This ensures every plugin behaves consistently.

---

# Plugin Structure

Every plugin should contain:

* Metadata
* Installer
* Configuration
* Database Migrations (if required)
* API
* Services
* UI Components
* Documentation

Each plugin should be self-contained and should not require manual project modifications.

---

# CLI Philosophy

Developers should never manually edit project files to install a feature.

Instead, they use a CLI.

Example:

```
lms add payment
```

The CLI automatically:

* Downloads or links the plugin
* Installs dependencies
* Registers the plugin
* Adds configuration entries
* Runs database migrations
* Generates required files
* Validates dependencies

The installation should finish with minimal manual work.

---

# Plugin Installer Responsibilities

Every plugin includes an installer that tells the CLI how to configure the project.

Possible installer tasks include:

* Register plugin
* Install dependencies
* Add environment variables
* Create configuration
* Run migrations
* Register routes
* Register permissions
* Register admin menu
* Generate types
* Validate dependencies

The installer is responsible for setup, not the business logic.

---

# Plugin Dependencies

Plugins may depend on other plugins.

Example:

Certificate

Depends on:

* Course

Payment Gateway

Depends on:

* Payment Module

Quiz

Depends on:

* Course

The CLI should prevent installing plugins with missing dependencies.

---

# Core vs Plugins

Core responsibilities:

* Authentication
* Users
* Roles
* Plugin Manager
* Event Bus
* Configuration
* Database
* CLI

Plugins:

* Course
* Quiz
* Assignment
* Payment
* Certificate
* Analytics
* Chat
* Attendance
* Live Classes

The core should not know how a plugin works internally.

---

# Service Registry

The core exposes registries for extension points.

Examples:

* Payment Providers
* Storage Providers
* Email Providers
* Notification Providers
* Authentication Providers
* AI Providers

Plugins register themselves with the appropriate registry.

The rest of the application communicates only with the registry, never with a specific provider.

---

# Example: Payment System

The Payment feature is itself a plugin.

However, payment gateways are separate provider plugins.

Structure:

```
Payment Module
│
├── Payment Service
├── Checkout
├── Orders
├── Registry
│
├── Razorpay Provider
├── Stripe Provider
├── PayPal Provider
└── PhonePe Provider
```

The application only interacts with the Payment Module.

The Payment Module selects the configured provider.

Example checkout flow:

```
Student purchases a course

↓

Checkout Service

↓

Payment Module

↓

Selected Provider

↓

Payment Gateway

↓

Payment Completed

↓

Order Updated

↓

Course Access Granted
```

If the administrator changes the provider from Razorpay to Stripe, the checkout flow remains exactly the same.

Only the provider changes.

No other feature requires modification.

---

# Plugin Marketplace (Future)

The architecture should support a plugin marketplace.

Possible examples:

* Payment Plugins
* Storage Plugins
* AI Plugins
* Analytics Plugins
* CRM Integrations
* Video Providers
* Notification Providers

Installing a plugin should require only a single CLI command.

---

# Future CLI Commands

Examples:

```
lms add payment
```

```
lms add payment-razorpay
```

```
lms remove payment-razorpay
```

```
lms update payment
```

```
lms list
```

```
lms create plugin payment-phonepe
```

```
lms enable payment-razorpay
```

```
lms disable payment-razorpay
```

The CLI should become the standard way to manage plugins.

---

# Benefits

* Modular architecture
* Easier maintenance
* Better code separation
* Faster feature development
* Third-party extensibility
* One-command installation
* Easier testing
* Reduced coupling
* Future marketplace support
* Better developer experience

---
