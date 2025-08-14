# Customization Patterns - Complete Guide

> **Comprehensive reference for app development, custom fields, hooks system, and extensibility patterns**

## Table of Contents

- [App Development Architecture](#app-development-architecture)
- [Hooks System](#hooks-system)
- [Custom Fields and Customizations](#custom-fields-and-customizations)
- [Fixture Management](#fixture-management)
- [App Installation and Setup](#app-installation-and-setup)
- [Extension Patterns](#extension-patterns)
- [Module Development](#module-development)
- [Client-Side Customizations](#client-side-customizations)
- [Server Script Integration](#server-script-integration)
- [Migration and Updates](#migration-and-updates)
- [Integration Patterns](#integration-patterns)
- [Best Practices](#best-practices)

## App Development Architecture

### App Structure Overview

```
library_management/
├── library_management/
│   ├── __init__.py
│   ├── hooks.py                    # App configuration and hooks
│   ├── modules.txt                 # List of modules
│   ├── patches/                    # Database patches
│   │   ├── __init__.py
│   │   └── v1_0/
│   │       ├── __init__.py
│   │       └── add_isbn_index.py
│   ├── fixtures/                   # Data fixtures
│   │   ├── custom_field.json
│   │   ├── workflow.json
│   │   └── role.json
│   ├── library_management/         # Main module
│   │   ├── __init__.py
│   │   └── doctype/
│   │       ├── library_book/       # DocType folder
│   │       │   ├── __init__.py
│   │       │   ├── library_book.py
│   │       │   ├── library_book.json
│   │       │   ├── library_book.js
│   │       │   ├── test_library_book.py
│   │       │   └── library_book_list.js
│   │       └── library_member/
│   ├── templates/                  # Web templates
│   │   ├── includes/
│   │   ├── pages/
│   │   └── generators/
│   ├── public/                     # Static assets
│   │   ├── css/
│   │   ├── js/
│   │   └── images/
│   ├── www/                        # Web pages
│   │   ├── library/
│   │   │   ├── index.py
│   │   │   └── index.html
│   │   └── book-catalog.py
│   └── config/                     # Configuration
│       ├── desktop.py              # Desktop icons
│       ├── docs.py                 # Documentation config
│       └── library_management.py   # Module config
├── requirements.txt                # Python dependencies
├── package.json                    # JavaScript dependencies  
├── setup.py                        # App installation
├── pyproject.toml                  # Python project config
├── README.md
└── license.txt
```

### App Initialization

```python
# library_management/__init__.py
__version__ = "1.0.0"

# Basic app information
def get_app_info():
    """Return app information"""
    return {
        "app_name": "library_management",
        "app_title": "Library Management System",
        "app_version": __version__,
        "app_description": "Complete library management solution",
        "app_publisher": "Your Company",
        "app_license": "MIT"
    }

def boot_session(bootinfo):
    """Boot session hook for adding custom data to bootinfo"""
    
    # Add custom settings to bootinfo
    bootinfo.library_settings = frappe.get_single("Library Settings").as_dict()
    
    # Add user-specific data
    if frappe.session.user != "Guest":
        member = frappe.db.get_value("Library Member", {"user": frappe.session.user}, "name")
        bootinfo.library_member = member
        
        if member:
            # Add member's current issues
            bootinfo.my_current_issues = frappe.get_all("Library Issue",
                filters={"member": member, "return_date": ["is", "not set"]},
                fields=["name", "book", "issue_date", "due_date"]
            )

def get_context_for_dev():
    """Add development context (only in dev mode)"""
    if frappe.conf.developer_mode:
        return {
            "show_debug_toolbar": True,
            "allow_app_reload": True
        }
    return {}
```

### Setup Configuration

```python
# setup.py
from setuptools import setup, find_packages

with open("requirements.txt") as f:
    install_requires = f.read().strip().split("\n")

# Get version from library_management/__init__.py
from library_management import __version__ as version

setup(
    name="library_management",
    version=version,
    description="Complete library management solution for Frappe",
    author="Your Company",
    author_email="info@yourcompany.com",
    packages=find_packages(),
    zip_safe=False,
    include_package_data=True,
    install_requires=install_requires,
    python_requires=">=3.8",
    classifiers=[
        "Development Status :: 4 - Beta",
        "Environment :: Web Environment",
        "Framework :: Frappe",
        "Intended Audience :: Developers",
        "License :: OSI Approved :: MIT License",
        "Operating System :: OS Independent",
        "Programming Language :: Python :: 3",
        "Topic :: Internet :: WWW/HTTP",
        "Topic :: Software Development :: Libraries :: Application Frameworks",
    ]
)
```

## Hooks System

### Comprehensive Hooks Configuration

Based on analysis of `frappe/hooks.py:15-100`, here's a comprehensive hooks setup:

```python
# library_management/hooks.py
import os
from . import __version__ as app_version

# App Information
app_name = "library_management"
app_title = "Library Management"
app_publisher = "Your Company"
app_description = "Complete library management system with book cataloging, member management, and circulation tracking"
app_license = "MIT"
app_logo_url = "/assets/library_management/images/library-logo.svg"
app_color = "#2E8B57"  # Sea green
required_apps = ["frappe"]

# Version and Compatibility
develop_version = "1.x.x-develop"
app_version = app_version

# Installation Hooks
before_install = "library_management.setup.install.before_install"
after_install = "library_management.setup.install.after_install"
before_uninstall = "library_management.setup.install.before_uninstall"
after_migrate = "library_management.setup.install.after_migrate"

# JavaScript and CSS Assets
app_include_js = [
    "/assets/library_management/js/library_common.js",
    "/assets/library_management/js/barcode_scanner.js"
]

app_include_css = [
    "/assets/library_management/css/library_theme.css"
]

# Web Assets (for website/portal pages)
web_include_js = [
    "/assets/library_management/js/library_portal.js"
]

web_include_css = [
    "/assets/library_management/css/library_portal.css"
]

# DocType-specific JavaScript and CSS
doctype_js = {
    "Library Book": "public/js/library_book.js",
    "Library Member": "public/js/library_member.js",
    "Library Issue": "public/js/library_issue.js"
}

doctype_css = {
    "Library Book": "public/css/library_book.css"
}

# List View JavaScript
doctype_list_js = {
    "Library Book": "public/js/library_book_list.js",
    "Library Issue": "public/js/library_issue_list.js"
}

# Calendar JavaScript
doctype_calendar_js = {
    "Library Event": "public/js/library_event_calendar.js"
}

# Tree JavaScript for hierarchical DocTypes
doctype_tree_js = {
    "Book Category": "public/js/book_category_tree.js"
}

# Document Event Hooks
doc_events = {
    "Library Book": {
        "before_insert": "library_management.library_management.doctype.library_book.library_book.before_insert",
        "after_insert": "library_management.library_management.doctype.library_book.library_book.after_insert",
        "before_save": "library_management.library_management.doctype.library_book.library_book.before_save",
        "on_update": "library_management.library_management.doctype.library_book.library_book.on_update",
        "before_submit": "library_management.library_management.doctype.library_book.library_book.before_submit",
        "on_submit": "library_management.library_management.doctype.library_book.library_book.on_submit",
        "on_cancel": "library_management.library_management.doctype.library_book.library_book.on_cancel",
        "before_trash": "library_management.library_management.doctype.library_book.library_book.before_trash",
        "after_delete": "library_management.library_management.doctype.library_book.library_book.after_delete"
    },
    "Library Member": {
        "validate": "library_management.library_management.doctype.library_member.library_member.validate",
        "on_update": "library_management.utils.member_utils.update_member_portal_access"
    },
    "User": {
        # Hook into User creation to create Library Member
        "after_insert": "library_management.utils.member_utils.create_library_member_from_user"
    }
}

# Scheduled Tasks (Cron Jobs)
scheduler_events = {
    "all": [
        "library_management.tasks.all.execute_all_tasks"
    ],
    "daily": [
        "library_management.tasks.daily.send_overdue_notifications",
        "library_management.tasks.daily.update_book_popularity_scores",
        "library_management.tasks.daily.cleanup_temporary_data"
    ],
    "hourly": [
        "library_management.tasks.hourly.sync_external_catalog",
        "library_management.tasks.hourly.update_real_time_stats"
    ],
    "weekly": [
        "library_management.tasks.weekly.generate_circulation_reports",
        "library_management.tasks.weekly.cleanup_old_logs"
    ],
    "monthly": [
        "library_management.tasks.monthly.generate_monthly_reports",
        "library_management.tasks.monthly.archive_old_data"
    ],
    "cron": {
        # Custom cron expressions
        "0 0 * * 0": [  # Every Sunday at midnight
            "library_management.tasks.weekly.weekly_maintenance"
        ],
        "0 6 1 * *": [  # First day of month at 6 AM
            "library_management.tasks.monthly.monthly_inventory_check"
        ]
    }
}

# Session Events
on_session_creation = [
    "library_management.utils.session.on_session_creation"
]

on_logout = [
    "library_management.utils.session.clear_library_session_data"
]

# User Events
on_login = [
    "library_management.utils.auth.on_user_login"
]

# Boot Session Hook
boot_session = "library_management.boot.boot_session"

# Email Events
email_append_to = [
    "Library Issue",  # Allow email replies to Library Issue
    "Library Reservation"
]

# Permission Hooks
permission_query_conditions = {
    "Library Book": "library_management.library_management.doctype.library_book.library_book.get_permission_query_conditions",
    "Library Issue": "library_management.library_management.doctype.library_issue.library_issue.get_permission_query_conditions"
}

has_permission = {
    "Library Book": "library_management.library_management.doctype.library_book.library_book.has_permission",
    "Library Issue": "library_management.library_management.doctype.library_issue.library_issue.has_permission"
}

# Website and Portal
website_route_rules = [
    {"from_route": "/library", "to_route": "Library Portal"},
    {"from_route": "/library/catalog", "to_route": "Book Catalog"},
    {"from_route": "/library/search/<query>", "to_route": "Search Results"},
    {"from_route": "/library/book/<book_id>", "to_route": "Book Details"}
]

portal_menu_items = [
    {
        "title": "My Library",
        "route": "/library",
        "reference_doctype": "Library Member",
        "role": "Library Member"
    },
    {
        "title": "Book Catalog", 
        "route": "/library/catalog",
        "role": "Library Member"
    }
]

# Standard portal pages
standard_portal_menu_items = [
    {
        "title": "Issues",
        "route": "/library/issues", 
        "reference_doctype": "Library Issue",
        "role": "Library Member"
    }
]

# Website Context Processors
website_context = {
    "favicon": "/assets/library_management/images/favicon.ico",
    "splash_image": "/assets/library_management/images/library-splash.jpg"
}

# Fixtures - Data to be imported on installation
fixtures = [
    {"doctype": "Role", "filters": {"name": ["in", ["Librarian", "Library Member", "Library Manager"]]}},
    {"doctype": "Custom Field", "filters": {"dt": ["in", ["User", "Customer"]]}},
    {"doctype": "Property Setter", "filters": {"doc_type": ["in", ["Library Book", "Library Member"]]}},
    {"doctype": "Workflow", "filters": {"name": ["in", ["Book Approval Workflow"]]}},
    {"doctype": "Print Format", "filters": {"doc_type": ["in", ["Library Issue", "Library Card"]]}},
    {"doctype": "Report", "filters": {"ref_doctype": ["in", ["Library Book", "Library Issue"]]}},
    {
        "doctype": "Translation",
        "filters": {"language": ["in", ["es", "fr", "de", "pt"]], "source_text": ["like", "Library%"]}
    }
]

# Override Hooks - Replace standard functionality
override_doctype_class = {
    # Override User DocType to add library-specific functionality
    "User": "library_management.overrides.user.CustomUser"
}

override_whitelisted_methods = {
    # Override standard search to include library-specific logic
    "frappe.desk.search.search_link": "library_management.overrides.search.library_search_link"
}

# Report Hooks
standard_queries = {
    "Library Book": "library_management.queries.library_book_query"
}

# Notification Hooks
notification_config = "library_management.notifications.get_notification_config"

# Translation
default_mail_footer = "library_management.utils.mail.get_mail_footer"

# Jinja Environment
jinja = {
    "methods": [
        "library_management.utils.jinja_methods.get_book_cover",
        "library_management.utils.jinja_methods.format_isbn",
        "library_management.utils.jinja_methods.get_member_stats"
    ],
    "filters": [
        "library_management.utils.jinja_filters.book_title_case",
        "library_management.utils.jinja_filters.duration_in_words"
    ]
}

# REST API Hooks
api_hooks = {
    "library/books": {
        "GET": "library_management.api.books.get_books",
        "POST": "library_management.api.books.create_book"
    },
    "library/search": {
        "GET": "library_management.api.search.search_catalog"
    }
}

# Background Jobs
background_job_hooks = {
    "library_book_sync": "library_management.background_jobs.sync_external_catalog",
    "send_due_date_reminders": "library_management.background_jobs.send_reminders"
}

# Integration Hooks
integrations = {
    "stripe": {
        "payment_success": "library_management.integrations.stripe.on_payment_success"
    },
    "zapier": {
        "trigger_events": [
            "Library Issue.on_submit",
            "Library Book.after_insert"
        ]
    }
}

# Development Hooks (only in developer mode)
if os.environ.get("FRAPPE_DEV_SERVER"):
    after_app_install = "library_management.setup.development.setup_dev_environment"
    before_migrate = "library_management.setup.development.backup_dev_data"
```

### Document Event Implementations

```python
# library_management/library_management/doctype/library_book/library_book.py

def before_insert(doc, method):
    """Execute before inserting new book"""
    
    # Auto-generate book code
    if not doc.book_code:
        doc.book_code = generate_book_code(doc)
        
    # Validate ISBN
    if doc.isbn:
        validate_isbn_format(doc.isbn)
        check_isbn_duplicates(doc.isbn, doc.name)
        
    # Set default values
    doc.status = doc.status or "Available"
    doc.date_added = frappe.utils.today()
    
    # Log activity
    frappe.get_doc({
        "doctype": "Activity Log",
        "subject": f"New book added: {doc.book_name}",
        "content": f"ISBN: {doc.isbn}, Category: {doc.category}",
        "reference_doctype": "Library Book",
        "reference_name": doc.name
    }).insert(ignore_permissions=True)

def after_insert(doc, method):
    """Execute after book is inserted"""
    
    # Update inventory counts
    update_category_counts(doc.category)
    
    # Send notification to librarians
    send_new_book_notification(doc)
    
    # Create catalog entry for search
    create_search_index_entry(doc)
    
    # Trigger external API sync
    frappe.enqueue("library_management.api.external.sync_new_book", 
                  book_id=doc.name, queue="long")

def validate(doc, method):
    """Validation logic for book"""
    
    # Validate publication year
    current_year = frappe.utils.now_datetime().year
    if doc.publication_year and doc.publication_year > current_year + 1:
        frappe.throw(_("Publication year cannot be in the future"))
        
    # Validate category exists
    if doc.category and not frappe.db.exists("Book Category", doc.category):
        frappe.throw(_("Invalid book category"))
        
    # Validate price
    if doc.price and doc.price < 0:
        frappe.throw(_("Price cannot be negative"))

def on_update(doc, method):
    """Execute when book is updated"""
    
    # Track field changes
    if doc.has_value_changed("status"):
        log_status_change(doc)
        
    if doc.has_value_changed("location"):
        log_location_change(doc)
        
    # Update search index if relevant fields changed  
    search_fields = ["book_name", "author", "isbn", "category", "keywords"]
    if any(doc.has_value_changed(field) for field in search_fields):
        update_search_index_entry(doc)
        
    # Clear related caches
    frappe.cache.delete_key("book_stats")
    frappe.cache.delete_key(f"book_details_{doc.name}")

def before_submit(doc, method):
    """Before submitting book (if submittable)"""
    
    if doc.status != "Available":
        frappe.throw(_("Only available books can be submitted"))
        
    # Final validation
    if not doc.isbn:
        frappe.throw(_("ISBN is required for submission"))

def on_submit(doc, method):
    """After book submission"""
    
    # Create initial activity log
    frappe.get_doc({
        "doctype": "Library Activity",
        "activity_type": "Book Catalogued",
        "book": doc.name,
        "activity_date": frappe.utils.now(),
        "details": f"Book {doc.book_name} officially added to catalog"
    }).insert(ignore_permissions=True)

def on_cancel(doc, method):
    """When book is cancelled"""
    
    # Check if book has active issues
    active_issues = frappe.get_all("Library Issue",
        filters={"book": doc.name, "return_date": ["is", "not set"]})
        
    if active_issues:
        frappe.throw(_("Cannot cancel book with active issues"))
        
    # Update status
    doc.db_set("status", "Cancelled")

def before_trash(doc, method):
    """Before deleting book"""
    
    # Check dependencies
    has_issues = frappe.db.exists("Library Issue", {"book": doc.name})
    if has_issues:
        frappe.throw(_("Cannot delete book with historical issues. Archive instead."))
        
    has_reservations = frappe.db.exists("Library Reservation", {"book": doc.name})
    if has_reservations:
        frappe.throw(_("Cannot delete book with reservations"))

def after_delete(doc, method):
    """After book is deleted"""
    
    # Clean up related data
    frappe.db.delete("Library Activity", {"book": doc.name})
    frappe.db.delete("Book Review", {"book": doc.name})
    
    # Remove from search index
    remove_from_search_index(doc.name)
    
    # Update category counts
    update_category_counts(doc.category, increment=False)
```

### Scheduled Task Implementation

```python
# library_management/tasks/daily.py

def send_overdue_notifications():
    """Send notifications for overdue books"""
    
    from frappe.utils import add_days, today
    
    # Get overdue issues
    overdue_issues = frappe.get_all("Library Issue",
        filters={
            "return_date": ["is", "not set"],
            "due_date": ["<", today()]
        },
        fields=["name", "member", "book", "due_date", "issue_date"]
    )
    
    for issue in overdue_issues:
        days_overdue = (frappe.utils.getdate() - issue.due_date).days
        
        # Get member details
        member = frappe.get_doc("Library Member", issue.member)
        book = frappe.get_doc("Library Book", issue.book)
        
        # Calculate late fee
        late_fee = calculate_late_fee(days_overdue)
        
        # Send email notification
        frappe.sendmail(
            recipients=[member.email],
            subject=f"Overdue Book: {book.book_name}",
            template="overdue_notification",
            args={
                "member_name": member.full_name,
                "book_name": book.book_name,
                "due_date": frappe.utils.formatdate(issue.due_date),
                "days_overdue": days_overdue,
                "late_fee": late_fee,
                "return_url": f"{frappe.utils.get_url()}/library/return/{issue.name}"
            }
        )
        
        # Log notification
        frappe.get_doc({
            "doctype": "Communication",
            "communication_type": "Automated Email",
            "subject": f"Overdue notification sent to {member.full_name}",
            "content": f"Book: {book.book_name}, Days overdue: {days_overdue}",
            "reference_doctype": "Library Issue",
            "reference_name": issue.name
        }).insert(ignore_permissions=True)

def update_book_popularity_scores():
    """Update popularity scores based on circulation data"""
    
    # Calculate popularity scores based on various factors
    popularity_query = """
        UPDATE `tabLibrary Book` 
        SET popularity_score = (
            SELECT COALESCE(
                (issue_count * 0.4) + 
                (rating * 10 * 0.3) + 
                (recent_issues * 0.3), 0
            )
            FROM (
                SELECT 
                    b.name,
                    COUNT(i.name) as issue_count,
                    COALESCE(AVG(r.rating), 0) as rating,
                    COUNT(CASE 
                        WHEN i.issue_date >= DATE_SUB(NOW(), INTERVAL 30 DAY) 
                        THEN 1 END) as recent_issues
                FROM `tabLibrary Book` b
                LEFT JOIN `tabLibrary Issue` i ON b.name = i.book
                LEFT JOIN `tabBook Review` r ON b.name = r.book
                WHERE b.name = `tabLibrary Book`.name
                GROUP BY b.name
            ) stats
        )
        WHERE status = 'Available'
    """
    
    frappe.db.sql(popularity_query)
    
    # Update trending books list
    update_trending_books_cache()

def cleanup_temporary_data():
    """Clean up temporary and expired data"""
    
    # Remove expired search cache
    frappe.db.delete("Search Cache", {
        "creation": ["<", frappe.utils.add_days(frappe.utils.today(), -7)]
    })
    
    # Clean up old activity logs (keep last 90 days)
    frappe.db.delete("Library Activity", {
        "activity_date": ["<", frappe.utils.add_days(frappe.utils.today(), -90)]
    })
    
    # Remove old session data
    frappe.db.delete("Library Session", {
        "modified": ["<", frappe.utils.add_days(frappe.utils.today(), -30)]
    })
    
    frappe.db.commit()
```

## Custom Fields and Customizations

### Programmatic Custom Field Creation

Based on analysis of `frappe/custom/doctype/custom_field/custom_field.py:15-100`:

```python
# library_management/setup/custom_fields.py

def create_library_custom_fields():
    """Create custom fields for library management"""
    
    custom_fields = {
        # Add library-related fields to User
        "User": [
            {
                "fieldname": "library_member_id",
                "label": "Library Member ID", 
                "fieldtype": "Link",
                "options": "Library Member",
                "insert_after": "email",
                "read_only": 1,
                "description": "Auto-linked library member record"
            },
            {
                "fieldname": "library_preferences",
                "label": "Library Preferences",
                "fieldtype": "Section Break",
                "insert_after": "library_member_id",
                "collapsible": 1
            },
            {
                "fieldname": "preferred_categories",
                "label": "Preferred Book Categories",
                "fieldtype": "Table MultiSelect",
                "options": "Book Category",
                "insert_after": "library_preferences"
            },
            {
                "fieldname": "notification_preferences", 
                "label": "Email Notifications",
                "fieldtype": "Select",
                "options": "All\nDue Date Reminders Only\nOverdue Only\nNone",
                "default": "All",
                "insert_after": "preferred_categories"
            }
        ],
        
        # Add customer integration to Library Member
        "Library Member": [
            {
                "fieldname": "customer_link",
                "label": "Customer",
                "fieldtype": "Link", 
                "options": "Customer",
                "insert_after": "email",
                "description": "Link to Customer record for billing"
            },
            {
                "fieldname": "membership_details",
                "label": "Membership Details",
                "fieldtype": "Section Break",
                "insert_after": "customer_link"
            },
            {
                "fieldname": "membership_start_date",
                "label": "Membership Start Date",
                "fieldtype": "Date",
                "insert_after": "membership_details",
                "reqd": 1,
                "default": "Today"
            },
            {
                "fieldname": "membership_end_date",
                "label": "Membership End Date", 
                "fieldtype": "Date",
                "insert_after": "membership_start_date"
            },
            {
                "fieldname": "max_books_allowed",
                "label": "Maximum Books Allowed",
                "fieldtype": "Int",
                "insert_after": "membership_end_date",
                "default": 3
            }
        ],
        
        # Add inventory fields to Item (if using ERPNext)
        "Item": [
            {
                "fieldname": "is_library_book",
                "label": "Is Library Book",
                "fieldtype": "Check",
                "insert_after": "item_group",
                "description": "Check if this item represents a library book"
            },
            {
                "fieldname": "library_details",
                "label": "Library Details",
                "fieldtype": "Section Break", 
                "insert_after": "is_library_book",
                "depends_on": "is_library_book",
                "collapsible": 1
            },
            {
                "fieldname": "isbn",
                "label": "ISBN",
                "fieldtype": "Data",
                "insert_after": "library_details",
                "depends_on": "is_library_book"
            },
            {
                "fieldname": "author",
                "label": "Author",
                "fieldtype": "Data",
                "insert_after": "isbn",
                "depends_on": "is_library_book"
            },
            {
                "fieldname": "publication_year",
                "label": "Publication Year",
                "fieldtype": "Int",
                "insert_after": "author",
                "depends_on": "is_library_book"
            }
        ]
    }
    
    # Create custom fields
    from frappe.custom.doctype.custom_field.custom_field import create_custom_fields
    create_custom_fields(custom_fields, update=True)

def create_conditional_custom_fields():
    """Create custom fields based on conditions"""
    
    # Only create ERPNext integration fields if ERPNext is installed
    if "erpnext" in frappe.get_installed_apps():
        erpnext_fields = {
            "Customer": [
                {
                    "fieldname": "is_library_member",
                    "label": "Is Library Member",
                    "fieldtype": "Check",
                    "insert_after": "customer_type"
                },
                {
                    "fieldname": "library_member",
                    "label": "Library Member",
                    "fieldtype": "Link",
                    "options": "Library Member", 
                    "insert_after": "is_library_member",
                    "depends_on": "is_library_member"
                }
            ],
            
            "Sales Invoice": [
                {
                    "fieldname": "library_late_fees",
                    "label": "Library Late Fees",
                    "fieldtype": "Section Break",
                    "insert_after": "taxes_and_charges"
                },
                {
                    "fieldname": "library_issues",
                    "label": "Related Library Issues",
                    "fieldtype": "Table",
                    "options": "Library Issue Reference",
                    "insert_after": "library_late_fees"
                }
            ]
        }
        
        from frappe.custom.doctype.custom_field.custom_field import create_custom_fields
        create_custom_fields(erpnext_fields, update=True)

def setup_property_setters():
    """Set up property setters for customization"""
    
    property_setters = [
        # Customize User form
        {
            "doctype": "Property Setter",
            "doctype_or_field": "DocType",
            "doc_type": "User",
            "property": "title_field",
            "value": "full_name",
            "property_type": "Data"
        },
        
        # Make email mandatory in Library Member
        {
            "doctype": "Property Setter",
            "doctype_or_field": "DocField", 
            "doc_type": "Library Member",
            "field_name": "email",
            "property": "reqd",
            "value": "1",
            "property_type": "Check"
        },
        
        # Set default print format
        {
            "doctype": "Property Setter",
            "doctype_or_field": "DocType",
            "doc_type": "Library Issue",
            "property": "default_print_format", 
            "value": "Library Issue Receipt",
            "property_type": "Data"
        }
    ]
    
    for prop in property_setters:
        if not frappe.db.exists("Property Setter", {
            "doc_type": prop["doc_type"],
            "property": prop["property"],
            "field_name": prop.get("field_name")
        }):
            frappe.get_doc(prop).insert(ignore_permissions=True)

def create_advanced_customizations():
    """Create advanced customizations with complex logic"""
    
    def create_dynamic_select_field():
        """Create select field with dynamic options"""
        
        frappe.get_doc({
            "doctype": "Custom Field",
            "dt": "Library Book",
            "fieldname": "reading_level",
            "label": "Reading Level",
            "fieldtype": "Select", 
            "options": get_reading_level_options(),
            "insert_after": "category",
            "description": "Age-appropriate reading level classification"
        }).insert(ignore_permissions=True, ignore_if_duplicate=True)
        
    def create_computed_field():
        """Create field with computed values"""
        
        frappe.get_doc({
            "doctype": "Custom Field",
            "dt": "Library Issue",
            "fieldname": "days_overdue",
            "label": "Days Overdue",
            "fieldtype": "Int",
            "read_only": 1,
            "insert_after": "due_date",
            "description": "Automatically calculated overdue days"
        }).insert(ignore_permissions=True, ignore_if_duplicate=True)
        
    def create_dependent_fields():
        """Create fields that depend on other field values"""
        
        # Field that shows only when membership type is Premium
        frappe.get_doc({
            "doctype": "Custom Field",
            "dt": "Library Member", 
            "fieldname": "premium_benefits",
            "label": "Premium Benefits",
            "fieldtype": "Text",
            "depends_on": 'eval:doc.membership_type=="Premium"',
            "insert_after": "membership_type",
            "description": "Benefits available to premium members"
        }).insert(ignore_permissions=True, ignore_if_duplicate=True)
        
    # Execute advanced customizations
    create_dynamic_select_field()
    create_computed_field()
    create_dependent_fields()

def get_reading_level_options():
    """Dynamically generate reading level options"""
    
    levels = [
        "Early Reader (Ages 3-5)",
        "Beginning Reader (Ages 5-7)", 
        "Developing Reader (Ages 7-9)",
        "Independent Reader (Ages 9-12)",
        "Young Adult (Ages 12-18)",
        "Adult"
    ]
    
    return "\n".join([""] + levels)  # Start with empty option
```

### Custom Field Management

```python
# library_management/utils/customization.py

class CustomFieldManager:
    """Manage custom fields programmatically"""
    
    def __init__(self):
        self.created_fields = []
        
    def add_field(self, doctype, field_dict):
        """Add a custom field with validation"""
        
        # Validate field definition
        self.validate_field_definition(field_dict)
        
        # Check if field already exists
        if self.field_exists(doctype, field_dict["fieldname"]):
            print(f"Field {field_dict['fieldname']} already exists in {doctype}")
            return
            
        try:
            custom_field = frappe.get_doc({
                "doctype": "Custom Field",
                "dt": doctype,
                **field_dict
            })
            
            custom_field.insert(ignore_permissions=True)
            self.created_fields.append((doctype, field_dict["fieldname"]))
            
            print(f"Created field {field_dict['fieldname']} in {doctype}")
            
        except Exception as e:
            print(f"Failed to create field {field_dict['fieldname']} in {doctype}: {e}")
            
    def validate_field_definition(self, field_dict):
        """Validate field definition before creation"""
        
        required_fields = ["fieldname", "label", "fieldtype"]
        
        for field in required_fields:
            if field not in field_dict:
                frappe.throw(f"Missing required field: {field}")
                
        # Validate fieldname
        if not field_dict["fieldname"].replace("_", "").isalnum():
            frappe.throw("Fieldname can only contain letters, numbers, and underscores")
            
        # Validate fieldtype
        valid_fieldtypes = frappe.get_meta("DocField").get_field("fieldtype").options.split("\n")
        if field_dict["fieldtype"] not in valid_fieldtypes:
            frappe.throw(f"Invalid fieldtype: {field_dict['fieldtype']}")
            
    def field_exists(self, doctype, fieldname):
        """Check if field already exists"""
        
        # Check in standard fields
        meta = frappe.get_meta(doctype)
        if meta.get_field(fieldname):
            return True
            
        # Check in custom fields
        if frappe.db.exists("Custom Field", {"dt": doctype, "fieldname": fieldname}):
            return True
            
        return False
        
    def remove_field(self, doctype, fieldname):
        """Remove a custom field"""
        
        custom_field = frappe.db.get_value("Custom Field", {
            "dt": doctype,
            "fieldname": fieldname
        }, "name")
        
        if custom_field:
            frappe.delete_doc("Custom Field", custom_field)
            print(f"Removed field {fieldname} from {doctype}")
        else:
            print(f"Field {fieldname} not found in {doctype}")
            
    def update_field_property(self, doctype, fieldname, property_name, value):
        """Update a field property"""
        
        custom_field = frappe.db.get_value("Custom Field", {
            "dt": doctype, 
            "fieldname": fieldname
        }, "name")
        
        if custom_field:
            frappe.db.set_value("Custom Field", custom_field, property_name, value)
            print(f"Updated {property_name} for {fieldname} in {doctype}")
        else:
            print(f"Field {fieldname} not found in {doctype}")
            
    def backup_customizations(self):
        """Backup all customizations to JSON"""
        
        customizations = {
            "custom_fields": [],
            "property_setters": [],
            "created_at": frappe.utils.now()
        }
        
        # Export custom fields
        custom_fields = frappe.get_all("Custom Field", 
            fields=["*"])
        customizations["custom_fields"] = custom_fields
        
        # Export property setters
        property_setters = frappe.get_all("Property Setter",
            fields=["*"])
        customizations["property_setters"] = property_setters
        
        # Save to file
        import json
        backup_file = f"library_customizations_{frappe.utils.now().replace(' ', '_')}.json"
        
        with open(backup_file, 'w') as f:
            json.dump(customizations, f, indent=2, default=str)
            
        print(f"Customizations backed up to {backup_file}")
        
    def restore_customizations(self, backup_file):
        """Restore customizations from backup"""
        
        import json
        
        with open(backup_file, 'r') as f:
            customizations = json.load(f)
            
        # Restore custom fields
        for field_data in customizations["custom_fields"]:
            if not self.field_exists(field_data["dt"], field_data["fieldname"]):
                try:
                    custom_field = frappe.get_doc({
                        "doctype": "Custom Field",
                        **field_data
                    })
                    custom_field.insert(ignore_permissions=True)
                except Exception as e:
                    print(f"Failed to restore field {field_data['fieldname']}: {e}")
                    
        print("Customizations restored successfully")

# Usage example
field_manager = CustomFieldManager()

field_manager.add_field("Library Book", {
    "fieldname": "digital_copy_available",
    "label": "Digital Copy Available",
    "fieldtype": "Check",
    "insert_after": "status",
    "description": "Whether a digital copy is available for download"
})
```

## Fixture Management

### Fixture Creation and Management

Based on analysis of `frappe/utils/fixtures.py:12-80`:

```python
# library_management/setup/fixtures.py

def create_initial_data():
    """Create initial data for library management"""
    
    # Create roles
    create_library_roles()
    
    # Create default book categories
    create_book_categories()
    
    # Create library settings
    create_library_settings()
    
    # Create print formats
    create_print_formats()
    
    # Create workflows
    create_workflows()

def create_library_roles():
    """Create library-specific roles"""
    
    roles = [
        {
            "role_name": "Library Manager",
            "desk_access": 1,
            "home_page": "/desk#workspace/Library",
            "two_factor_auth": 0
        },
        {
            "role_name": "Librarian", 
            "desk_access": 1,
            "home_page": "/desk#List/Library Book",
            "two_factor_auth": 0
        },
        {
            "role_name": "Library Member",
            "desk_access": 0,  # Portal access only
            "home_page": "/library",
            "two_factor_auth": 0
        },
        {
            "role_name": "Library Assistant",
            "desk_access": 1,
            "home_page": "/desk#List/Library Issue",
            "two_factor_auth": 0
        }
    ]
    
    for role_data in roles:
        if not frappe.db.exists("Role", role_data["role_name"]):
            role = frappe.get_doc({
                "doctype": "Role",
                **role_data
            })
            role.insert(ignore_permissions=True)
            print(f"Created role: {role_data['role_name']}")

def create_book_categories():
    """Create default book categories"""
    
    categories = [
        {"category_name": "Fiction", "description": "Fictional literature"},
        {"category_name": "Non-Fiction", "description": "Non-fictional works"},
        {"category_name": "Science", "description": "Scientific publications"},
        {"category_name": "Technology", "description": "Technology and computing"},
        {"category_name": "History", "description": "Historical works"},
        {"category_name": "Biography", "description": "Biographical works"},
        {"category_name": "Children", "description": "Children's books"},
        {"category_name": "Reference", "description": "Reference materials"},
        {"category_name": "Textbook", "description": "Educational textbooks"},
        {"category_name": "Journal", "description": "Academic journals"}
    ]
    
    for category_data in categories:
        if not frappe.db.exists("Book Category", category_data["category_name"]):
            category = frappe.get_doc({
                "doctype": "Book Category",
                **category_data
            })
            category.insert(ignore_permissions=True)
            print(f"Created category: {category_data['category_name']}")

def create_library_settings():
    """Create library settings singleton"""
    
    if not frappe.db.exists("Library Settings", "Library Settings"):
        settings = frappe.get_doc({
            "doctype": "Library Settings",
            "loan_period": 14,  # Days
            "max_books_per_member": 5,
            "late_fee_per_day": 1.00,
            "reservation_period": 7,  # Days
            "enable_email_notifications": 1,
            "enable_sms_notifications": 0,
            "library_name": "Community Library",
            "library_address": "123 Library Street",
            "library_phone": "+1234567890",
            "library_email": "info@library.com",
            "operating_hours": "9:00 AM - 6:00 PM",
            "closed_days": "Sunday"
        })
        settings.insert(ignore_permissions=True)
        print("Created Library Settings")

def create_workflows():
    """Create library workflows"""
    
    # Book Approval Workflow
    book_workflow = {
        "workflow_name": "Book Approval",
        "document_type": "Library Book",
        "is_active": 1,
        "states": [
            {
                "state": "Draft",
                "doc_status": "0",
                "allow_edit": "Librarian",
                "is_optional_state": 1
            },
            {
                "state": "Pending Review",
                "doc_status": "0", 
                "allow_edit": "Library Manager"
            },
            {
                "state": "Approved",
                "doc_status": "1",
                "allow_edit": ""
            },
            {
                "state": "Rejected",
                "doc_status": "0",
                "allow_edit": "Library Manager"
            }
        ],
        "transitions": [
            {
                "state": "Draft",
                "action": "Submit for Review",
                "next_state": "Pending Review",
                "allowed": "Librarian",
                "allow_self_approval": 0
            },
            {
                "state": "Pending Review",
                "action": "Approve",
                "next_state": "Approved", 
                "allowed": "Library Manager",
                "allow_self_approval": 0
            },
            {
                "state": "Pending Review",
                "action": "Reject",
                "next_state": "Rejected",
                "allowed": "Library Manager",
                "allow_self_approval": 0
            },
            {
                "state": "Rejected",
                "action": "Resubmit",
                "next_state": "Pending Review",
                "allowed": "Librarian",
                "allow_self_approval": 0
            }
        ]
    }
    
    if not frappe.db.exists("Workflow", book_workflow["workflow_name"]):
        workflow = frappe.get_doc({
            "doctype": "Workflow",
            **book_workflow
        })
        workflow.insert(ignore_permissions=True)
        print(f"Created workflow: {book_workflow['workflow_name']}")

def export_custom_fixtures():
    """Export custom fixtures to JSON files"""
    
    fixtures_to_export = [
        {
            "doctype": "Role",
            "filters": {"name": ["like", "Library%"]},
            "filename": "library_roles.json"
        },
        {
            "doctype": "Book Category", 
            "filters": {},
            "filename": "book_categories.json"
        },
        {
            "doctype": "Library Settings",
            "filters": {},
            "filename": "library_settings.json"
        },
        {
            "doctype": "Custom Field",
            "filters": {"dt": ["in", ["User", "Customer", "Library Member"]]},
            "filename": "library_custom_fields.json"
        },
        {
            "doctype": "Print Format",
            "filters": {"doc_type": ["in", ["Library Issue", "Library Card"]]},
            "filename": "library_print_formats.json"
        }
    ]
    
    import os
    import json
    
    fixtures_dir = frappe.get_app_path("library_management", "fixtures")
    if not os.path.exists(fixtures_dir):
        os.makedirs(fixtures_dir)
        
    for fixture in fixtures_to_export:
        data = frappe.get_all(fixture["doctype"],
            filters=fixture.get("filters", {}),
            fields=["*"]
        )
        
        # Get complete document data
        complete_data = []
        for record in data:
            doc = frappe.get_doc(fixture["doctype"], record["name"])
            complete_data.append(doc.as_dict())
            
        # Export to file
        filepath = os.path.join(fixtures_dir, fixture["filename"])
        with open(filepath, 'w') as f:
            json.dump(complete_data, f, indent=2, default=str)
            
        print(f"Exported {len(complete_data)} {fixture['doctype']} records to {fixture['filename']}")

def import_sample_data():
    """Import sample data for testing/demo"""
    
    # Sample authors
    authors = [
        {"email": "jdoe@example.com", "first_name": "John", "last_name": "Doe"},
        {"email": "jsmith@example.com", "first_name": "Jane", "last_name": "Smith"},
        {"email": "bwilson@example.com", "first_name": "Bob", "last_name": "Wilson"}
    ]
    
    for author_data in authors:
        if not frappe.db.exists("Author", author_data["email"]):
            author = frappe.get_doc({
                "doctype": "Author",
                **author_data
            })
            author.insert(ignore_permissions=True)
            
    # Sample books
    books = [
        {
            "book_name": "Introduction to Python Programming",
            "author": "jdoe@example.com",
            "isbn": "978-0-123456-78-9",
            "category": "Technology",
            "publication_year": 2023,
            "price": 45.99
        },
        {
            "book_name": "History of Ancient Rome", 
            "author": "jsmith@example.com",
            "isbn": "978-0-987654-32-1",
            "category": "History",
            "publication_year": 2022,
            "price": 32.50
        },
        {
            "book_name": "Children's Science Adventures",
            "author": "bwilson@example.com", 
            "isbn": "978-0-555666-77-8",
            "category": "Children",
            "publication_year": 2023,
            "price": 18.95
        }
    ]
    
    for book_data in books:
        if not frappe.db.exists("Library Book", {"isbn": book_data["isbn"]}):
            book = frappe.get_doc({
                "doctype": "Library Book",
                **book_data
            })
            book.insert(ignore_permissions=True)
            print(f"Created sample book: {book_data['book_name']}")
            
    # Sample members
    members = [
        {
            "first_name": "Alice",
            "last_name": "Johnson", 
            "email": "alice.johnson@example.com",
            "membership_type": "Premium"
        },
        {
            "first_name": "Bob",
            "last_name": "Brown",
            "email": "bob.brown@example.com", 
            "membership_type": "Basic"
        }
    ]
    
    for member_data in members:
        if not frappe.db.exists("Library Member", {"email": member_data["email"]}):
            member = frappe.get_doc({
                "doctype": "Library Member",
                **member_data
            })
            member.insert(ignore_permissions=True)
            print(f"Created sample member: {member_data['first_name']} {member_data['last_name']}")
```

## App Installation and Setup

### Installation Lifecycle

```python
# library_management/setup/install.py

def before_install():
    """Execute before app installation"""
    
    print("Starting Library Management System installation...")
    
    # Check prerequisites
    check_prerequisites()
    
    # Validate system settings
    validate_system_settings()
    
    # Create required directories
    create_required_directories()

def check_prerequisites():
    """Check system prerequisites"""
    
    # Check Frappe version
    frappe_version = frappe.__version__
    min_version = "15.0.0"
    
    if frappe_version < min_version:
        frappe.throw(f"Library Management requires Frappe >= {min_version}, found {frappe_version}")
        
    # Check required Python packages
    required_packages = ["barcode", "qrcode", "PIL"]
    
    for package in required_packages:
        try:
            __import__(package)
        except ImportError:
            frappe.throw(f"Required Python package '{package}' is not installed. Run: pip install {package}")
            
    # Check database version
    db_version = frappe.db.sql("SELECT VERSION()")[0][0]
    print(f"Database version: {db_version}")

def validate_system_settings():
    """Validate system settings for library management"""
    
    # Check if required system settings are enabled
    required_settings = {
        "enable_scheduler": 1,
        "backup_limit": 3
    }
    
    for setting, required_value in required_settings.items():
        current_value = frappe.db.get_single_value("System Settings", setting)
        if current_value != required_value:
            print(f"Warning: {setting} should be set to {required_value}")

def create_required_directories():
    """Create required directories"""
    
    import os
    
    directories = [
        frappe.get_site_path("public", "files", "library"),
        frappe.get_site_path("private", "files", "library", "backups"),
        frappe.get_site_path("private", "files", "library", "imports"),
        frappe.get_site_path("private", "files", "library", "exports")
    ]
    
    for directory in directories:
        if not os.path.exists(directory):
            os.makedirs(directory)
            print(f"Created directory: {directory}")

def after_install():
    """Execute after app installation"""
    
    print("Completing Library Management System installation...")
    
    # Create initial data
    create_initial_data()
    
    # Set up custom fields
    setup_customizations()
    
    # Create default reports
    create_default_reports()
    
    # Set up permissions
    setup_permissions()
    
    # Create sample data in development
    if frappe.conf.developer_mode:
        create_sample_data()
        
    # Set up integrations
    setup_integrations()
    
    print("Library Management System installed successfully!")

def create_initial_data():
    """Create essential initial data"""
    
    from library_management.setup.fixtures import (
        create_library_roles,
        create_book_categories, 
        create_library_settings,
        create_workflows
    )
    
    create_library_roles()
    create_book_categories()
    create_library_settings()
    create_workflows()

def setup_customizations():
    """Set up customizations"""
    
    from library_management.setup.custom_fields import (
        create_library_custom_fields,
        create_conditional_custom_fields,
        setup_property_setters
    )
    
    create_library_custom_fields()
    create_conditional_custom_fields()
    setup_property_setters()

def create_default_reports():
    """Create default reports"""
    
    reports = [
        {
            "name": "Book Circulation Report",
            "ref_doctype": "Library Issue",
            "report_type": "Script Report",
            "module": "Library Management",
            "is_standard": "Yes"
        },
        {
            "name": "Overdue Books Report",
            "ref_doctype": "Library Issue", 
            "report_type": "Query Report",
            "module": "Library Management",
            "is_standard": "Yes"
        },
        {
            "name": "Member Activity Report",
            "ref_doctype": "Library Member",
            "report_type": "Script Report", 
            "module": "Library Management",
            "is_standard": "Yes"
        }
    ]
    
    for report_data in reports:
        if not frappe.db.exists("Report", report_data["name"]):
            report = frappe.get_doc({
                "doctype": "Report",
                **report_data
            })
            report.insert(ignore_permissions=True)
            print(f"Created report: {report_data['name']}")

def setup_permissions():
    """Set up role-based permissions"""
    
    permissions = [
        # Library Manager - full access
        {
            "role": "Library Manager",
            "doctypes": ["Library Book", "Library Member", "Library Issue", "Library Settings"],
            "permissions": {"read": 1, "write": 1, "create": 1, "delete": 1, "submit": 1, "cancel": 1}
        },
        
        # Librarian - operational access
        {
            "role": "Librarian", 
            "doctypes": ["Library Book", "Library Member", "Library Issue"],
            "permissions": {"read": 1, "write": 1, "create": 1, "submit": 1}
        },
        
        # Library Assistant - limited access
        {
            "role": "Library Assistant",
            "doctypes": ["Library Book", "Library Issue"],
            "permissions": {"read": 1, "write": 1, "create": 1}
        },
        
        # Library Member - self-service access
        {
            "role": "Library Member",
            "doctypes": ["Library Issue"],
            "permissions": {"read": 1},
            "conditions": {"if_owner": 1}
        }
    ]
    
    for perm_group in permissions:
        for doctype in perm_group["doctypes"]:
            # Create permission record
            perm_doc = {
                "doctype": "DocPerm",
                "parent": doctype,
                "parenttype": "DocType",
                "parentfield": "permissions",
                "role": perm_group["role"],
                "permlevel": 0
            }
            
            # Add permissions
            perm_doc.update(perm_group["permissions"])
            
            # Add conditions if any
            if "conditions" in perm_group:
                perm_doc.update(perm_group["conditions"])
                
            # Insert if doesn't exist
            if not frappe.db.exists("DocPerm", {
                "parent": doctype,
                "role": perm_group["role"]
            }):
                frappe.get_doc(perm_doc).insert(ignore_permissions=True)
                print(f"Created permission for {perm_group['role']} on {doctype}")

def setup_integrations():
    """Set up third-party integrations"""
    
    # Set up email templates
    create_email_templates()
    
    # Set up print formats
    create_print_formats()
    
    # Set up web forms
    create_web_forms()

def create_email_templates():
    """Create email templates"""
    
    templates = [
        {
            "name": "Library Welcome Email",
            "subject": "Welcome to {{ library_name }}!",
            "use_html": 1,
            "response": """
            <h2>Welcome to {{ library_name }}!</h2>
            <p>Dear {{ member_name }},</p>
            <p>Thank you for joining our library. Your membership details:</p>
            <ul>
                <li>Member ID: {{ member_id }}</li>
                <li>Membership Type: {{ membership_type }}</li>
                <li>Start Date: {{ membership_start_date }}</li>
            </ul>
            <p>You can access your account at: <a href="{{ portal_url }}">Library Portal</a></p>
            <p>Happy reading!</p>
            """
        },
        {
            "name": "Book Due Reminder",
            "subject": "Book Due Tomorrow: {{ book_name }}",
            "use_html": 1,
            "response": """
            <p>Dear {{ member_name }},</p>
            <p>This is a reminder that the following book is due tomorrow:</p>
            <p><strong>{{ book_name }}</strong><br>
            Due Date: {{ due_date }}<br>
            Issue Date: {{ issue_date }}</p>
            <p>Please return the book by the due date to avoid late fees.</p>
            <p><a href="{{ return_url }}">Return Book Online</a></p>
            """
        }
    ]
    
    for template_data in templates:
        if not frappe.db.exists("Email Template", template_data["name"]):
            template = frappe.get_doc({
                "doctype": "Email Template",
                **template_data
            })
            template.insert(ignore_permissions=True)
            print(f"Created email template: {template_data['name']}")

def before_uninstall():
    """Execute before app uninstall"""
    
    print("Preparing to uninstall Library Management System...")
    
    # Warn about data loss
    confirm = input("This will delete all library data. Continue? (yes/no): ")
    if confirm.lower() != "yes":
        frappe.throw("Uninstallation cancelled")
        
    # Backup data
    backup_library_data()
    
    # Clean up scheduled jobs
    cleanup_scheduled_jobs()

def backup_library_data():
    """Backup library data before uninstall"""
    
    import json
    import os
    from datetime import datetime
    
    backup_data = {
        "backup_date": datetime.now().isoformat(),
        "library_books": frappe.get_all("Library Book", fields=["*"]),
        "library_members": frappe.get_all("Library Member", fields=["*"]),
        "library_issues": frappe.get_all("Library Issue", fields=["*"]),
        "library_settings": frappe.get_single("Library Settings").as_dict()
    }
    
    backup_file = f"library_backup_{datetime.now().strftime('%Y%m%d_%H%M%S')}.json"
    backup_path = frappe.get_site_path("private", "files", backup_file)
    
    with open(backup_path, 'w') as f:
        json.dump(backup_data, f, indent=2, default=str)
        
    print(f"Library data backed up to: {backup_path}")

def cleanup_scheduled_jobs():
    """Clean up scheduled jobs"""
    
    # Remove scheduled jobs that were added by this app
    job_types = [
        "library_management.tasks.daily.send_overdue_notifications",
        "library_management.tasks.hourly.sync_external_catalog",
        "library_management.tasks.weekly.generate_reports"
    ]
    
    for job_type in job_types:
        frappe.db.delete("Scheduled Job Type", {"method": job_type})
        
    print("Cleaned up scheduled jobs")

def after_migrate():
    """Execute after app migration/update"""
    
    print("Running post-migration tasks...")
    
    # Update custom fields if needed
    update_custom_fields()
    
    # Sync fixtures
    from frappe.utils.fixtures import sync_fixtures
    sync_fixtures("library_management")
    
    # Run data patches
    run_data_patches()
    
    # Update permissions
    update_permissions()
    
    print("Migration completed successfully!")

def update_custom_fields():
    """Update custom fields after migration"""
    
    # Add new fields introduced in latest version
    new_fields = get_version_specific_fields()
    
    for doctype, fields in new_fields.items():
        for field_data in fields:
            if not frappe.db.exists("Custom Field", {
                "dt": doctype,
                "fieldname": field_data["fieldname"]
            }):
                custom_field = frappe.get_doc({
                    "doctype": "Custom Field",
                    "dt": doctype,
                    **field_data
                })
                custom_field.insert(ignore_permissions=True)
                print(f"Added new field: {field_data['fieldname']} to {doctype}")

def get_version_specific_fields():
    """Get fields to add based on app version"""
    
    current_version = frappe.get_attr("library_management.__version__")
    
    # Version 1.1.0 additions
    if current_version >= "1.1.0":
        return {
            "Library Book": [
                {
                    "fieldname": "qr_code",
                    "label": "QR Code",
                    "fieldtype": "Attach",
                    "insert_after": "isbn",
                    "description": "QR code for quick book identification"
                }
            ]
        }
        
    return {}

def run_data_patches():
    """Run data migration patches"""
    
    # Execute version-specific data patches
    patches = [
        "library_management.patches.v1_0.add_isbn_index",
        "library_management.patches.v1_1.migrate_book_codes"
    ]
    
    for patch in patches:
        try:
            frappe.get_attr(patch).execute()
            print(f"Executed patch: {patch}")
        except Exception as e:
            print(f"Failed to execute patch {patch}: {e}")

def update_permissions():
    """Update permissions after migration"""
    
    # Reload doctypes to get updated permissions
    doctypes = ["Library Book", "Library Member", "Library Issue", "Library Settings"]
    
    for doctype in doctypes:
        frappe.reload_doctype(doctype)
        
    # Clear permission cache
    frappe.cache.delete_key("doctypes_with_read_permissions")
    frappe.cache.delete_key("boot")
```

## Extension Patterns

### Plugin Architecture

```python
# library_management/core/plugin_manager.py

class PluginManager:
    """Manage library management plugins"""
    
    def __init__(self):
        self.plugins = {}
        self.hooks = {
            "before_book_issue": [],
            "after_book_return": [],
            "member_registration": [],
            "payment_processing": []
        }
        
    def register_plugin(self, plugin_name, plugin_class):
        """Register a plugin"""
        
        self.plugins[plugin_name] = plugin_class()
        
        # Register plugin hooks
        if hasattr(plugin_class, "register_hooks"):
            plugin_hooks = plugin_class().register_hooks()
            for hook_name, methods in plugin_hooks.items():
                if hook_name in self.hooks:
                    self.hooks[hook_name].extend(methods)
                    
        print(f"Registered plugin: {plugin_name}")
        
    def execute_hook(self, hook_name, *args, **kwargs):
        """Execute all methods for a hook"""
        
        results = []
        
        for method in self.hooks.get(hook_name, []):
            try:
                result = method(*args, **kwargs)
                results.append(result)
            except Exception as e:
                frappe.log_error(f"Plugin hook error: {hook_name} - {e}")
                
        return results
        
    def get_plugin(self, plugin_name):
        """Get plugin instance"""
        return self.plugins.get(plugin_name)

# Base plugin class
class BaseLibraryPlugin:
    """Base class for library plugins"""
    
    def __init__(self):
        self.name = self.__class__.__name__
        
    def register_hooks(self):
        """Override to register plugin hooks"""
        return {}
        
    def install(self):
        """Plugin installation logic"""
        pass
        
    def uninstall(self):
        """Plugin uninstallation logic"""
        pass

# Example: Barcode scanning plugin
class BarcodeScannerPlugin(BaseLibraryPlugin):
    """Plugin for barcode scanning functionality"""
    
    def register_hooks(self):
        return {
            "before_book_issue": [self.scan_book_barcode],
            "after_book_return": [self.update_book_location]
        }
        
    def scan_book_barcode(self, issue_doc):
        """Scan barcode before book issue"""
        
        if issue_doc.book and not issue_doc.barcode_scanned:
            # Simulate barcode scanning
            book = frappe.get_doc("Library Book", issue_doc.book)
            
            if book.barcode:
                # Validate barcode
                if self.validate_barcode(book.barcode):
                    issue_doc.barcode_scanned = 1
                    issue_doc.scanned_at = frappe.utils.now()
                else:
                    frappe.throw("Invalid barcode scanned")
                    
    def update_book_location(self, issue_doc):
        """Update book location after return"""
        
        book = frappe.get_doc("Library Book", issue_doc.book)
        book.current_location = "Returned - Needs Shelving"
        book.save()
        
    def validate_barcode(self, barcode):
        """Validate barcode format"""
        import re
        return bool(re.match(r'^\d{10,13}$', barcode))

# Example: Fine calculation plugin
class FineCalculatorPlugin(BaseLibraryPlugin):
    """Plugin for fine calculation"""
    
    def register_hooks(self):
        return {
            "after_book_return": [self.calculate_fine]
        }
        
    def calculate_fine(self, issue_doc):
        """Calculate fine for overdue books"""
        
        if issue_doc.return_date and issue_doc.due_date:
            return_date = frappe.utils.getdate(issue_doc.return_date)
            due_date = frappe.utils.getdate(issue_doc.due_date)
            
            if return_date > due_date:
                days_overdue = (return_date - due_date).days
                
                # Get fine rate from settings
                fine_rate = frappe.db.get_single_value("Library Settings", "late_fee_per_day")
                total_fine = days_overdue * (fine_rate or 1.0)
                
                # Create fine record
                if total_fine > 0:
                    fine = frappe.get_doc({
                        "doctype": "Library Fine",
                        "member": issue_doc.member,
                        "issue": issue_doc.name,
                        "amount": total_fine,
                        "reason": f"Late return - {days_overdue} days overdue",
                        "fine_date": frappe.utils.today()
                    })
                    fine.insert(ignore_permissions=True)
                    
                    return {"fine_amount": total_fine, "days_overdue": days_overdue}

# Plugin registration
plugin_manager = PluginManager()
plugin_manager.register_plugin("barcode_scanner", BarcodeScannerPlugin)
plugin_manager.register_plugin("fine_calculator", FineCalculatorPlugin)
```

### API Extension Patterns

```python
# library_management/api/extensions.py

def extend_book_api():
    """Extend book API with custom endpoints"""
    
    @frappe.whitelist()
    def search_books_advanced(query, filters=None, page=1, limit=20):
        """Advanced book search with filtering"""
        
        # Build search query
        from frappe.model.db_query import DatabaseQuery
        
        db_query = DatabaseQuery("Library Book")
        
        # Add text search
        if query:
            conditions = [
                f"`tabLibrary Book`.book_name LIKE '%{query}%'",
                f"`tabLibrary Book`.author LIKE '%{query}%'",
                f"`tabLibrary Book`.isbn LIKE '%{query}%'"
            ]
            db_query.or_conditions.extend(conditions)
            
        # Add filters
        if filters:
            for field, value in filters.items():
                db_query.conditions.append(f"`tabLibrary Book`.{field} = '{value}'")
                
        # Execute search
        results = db_query.execute(
            fields=["name", "book_name", "author", "isbn", "status", "rating"],
            limit_start=(page - 1) * limit,
            limit_page_length=limit,
            order_by="popularity_score desc"
        )
        
        # Add additional data
        for book in results:
            book["cover_image"] = get_book_cover_url(book["name"])
            book["availability_status"] = check_book_availability(book["name"])
            
        return {
            "books": results,
            "total": len(results),
            "page": page,
            "has_more": len(results) == limit
        }
    
    @frappe.whitelist()
    def get_book_recommendations(member_id, limit=10):
        """Get personalized book recommendations"""
        
        member = frappe.get_doc("Library Member", member_id)
        
        # Get member's reading history
        reading_history = frappe.get_all("Library Issue",
            filters={"member": member_id},
            fields=["book"]
        )
        
        read_books = [issue["book"] for issue in reading_history]
        
        # Get member's preferred categories
        preferred_categories = member.get("preferred_categories", [])
        
        if not preferred_categories and read_books:
            # Infer preferences from reading history
            book_categories = frappe.get_all("Library Book",
                filters={"name": ["in", read_books]},
                fields=["category"],
                distinct=True
            )
            preferred_categories = [cat["category"] for cat in book_categories]
            
        # Find similar books
        recommendations = frappe.get_all("Library Book",
            filters={
                "category": ["in", preferred_categories],
                "name": ["not in", read_books],
                "status": "Available",
                "rating": [">=", 3.5]
            },
            fields=["name", "book_name", "author", "rating", "category"],
            order_by="rating desc, popularity_score desc",
            limit=limit
        )
        
        return {
            "member": member.full_name,
            "recommendations": recommendations,
            "based_on": preferred_categories
        }
    
    @frappe.whitelist(methods=["POST"])
    def rate_book(book_id, rating, review_text=None):
        """Rate and review a book"""
        
        member_id = get_current_member_id()
        if not member_id:
            frappe.throw("Only library members can rate books")
            
        # Check if user has borrowed this book
        has_borrowed = frappe.db.exists("Library Issue", {
            "member": member_id,
            "book": book_id
        })
        
        if not has_borrowed:
            frappe.throw("You can only rate books you have borrowed")
            
        # Create or update rating
        existing_rating = frappe.db.exists("Book Rating", {
            "book": book_id,
            "member": member_id
        })
        
        if existing_rating:
            rating_doc = frappe.get_doc("Book Rating", existing_rating)
            rating_doc.rating = rating
            rating_doc.review_text = review_text or rating_doc.review_text
        else:
            rating_doc = frappe.get_doc({
                "doctype": "Book Rating",
                "book": book_id,
                "member": member_id,
                "rating": rating,
                "review_text": review_text,
                "rating_date": frappe.utils.today()
            })
            
        rating_doc.save()
        
        # Update book's average rating
        update_book_average_rating(book_id)
        
        return {"message": "Rating saved successfully"}

def extend_member_portal():
    """Extend member portal functionality"""
    
    @frappe.whitelist()
    def get_member_dashboard():
        """Get member dashboard data"""
        
        member_id = get_current_member_id()
        if not member_id:
            return {}
            
        # Current issues
        current_issues = frappe.get_all("Library Issue",
            filters={"member": member_id, "return_date": ["is", "not set"]},
            fields=["name", "book", "issue_date", "due_date"],
            order_by="due_date"
        )
        
        # Add book details
        for issue in current_issues:
            book = frappe.get_doc("Library Book", issue["book"])
            issue["book_name"] = book.book_name
            issue["author"] = book.author
            issue["days_remaining"] = (frappe.utils.getdate(issue["due_date"]) - frappe.utils.today()).days
            
        # Reading statistics
        total_books_read = frappe.db.count("Library Issue", {
            "member": member_id,
            "return_date": ["is", "set"]
        })
        
        # Favorite categories
        favorite_categories = frappe.db.sql("""
            SELECT b.category, COUNT(*) as count
            FROM `tabLibrary Issue` i
            JOIN `tabLibrary Book` b ON i.book = b.name
            WHERE i.member = %s
            GROUP BY b.category
            ORDER BY count DESC
            LIMIT 5
        """, (member_id,), as_dict=True)
        
        return {
            "current_issues": current_issues,
            "total_books_read": total_books_read,
            "favorite_categories": favorite_categories,
            "overdue_count": len([i for i in current_issues if i["days_remaining"] < 0])
        }
    
    @frappe.whitelist()
    def reserve_book(book_id):
        """Reserve a book"""
        
        member_id = get_current_member_id()
        if not member_id:
            frappe.throw("Only library members can reserve books")
            
        # Check if book is available for reservation
        book = frappe.get_doc("Library Book", book_id)
        
        if book.status == "Available":
            frappe.throw("Book is available for immediate issue")
        elif book.status not in ["Issued", "Reserved"]:
            frappe.throw("Book is not available for reservation")
            
        # Check if member already has a reservation
        existing_reservation = frappe.db.exists("Library Reservation", {
            "member": member_id,
            "book": book_id,
            "status": "Active"
        })
        
        if existing_reservation:
            frappe.throw("You already have a reservation for this book")
            
        # Create reservation
        reservation = frappe.get_doc({
            "doctype": "Library Reservation",
            "member": member_id,
            "book": book_id,
            "reservation_date": frappe.utils.today(),
            "status": "Active"
        })
        reservation.insert()
        
        return {"message": "Book reserved successfully", "reservation_id": reservation.name}

def get_current_member_id():
    """Get current user's library member ID"""
    
    user = frappe.session.user
    if user == "Guest":
        return None
        
    return frappe.db.get_value("Library Member", {"user": user}, "name")
```

## Client-Side Customizations

### JavaScript Extension Patterns

```javascript
// library_management/public/js/library_common.js

// Global library utilities
frappe.library = {
    
    // Barcode scanning functionality
    scanBarcode: function(callback) {
        if ('BarcodeDetector' in window) {
            // Use native barcode detection if available
            this.useNativeBarcodeScanner(callback);
        } else {
            // Fallback to manual input
            this.showBarcodeInput(callback);
        }
    },
    
    useNativeBarcodeScanner: function(callback) {
        const barcodeDetector = new BarcodeDetector({
            formats: ['code_128', 'code_39', 'ean_13', 'ean_8']
        });
        
        // Create video element for camera
        const video = document.createElement('video');
        const canvas = document.createElement('canvas');
        const context = canvas.getContext('2d');
        
        navigator.mediaDevices.getUserMedia({ video: true })
            .then(stream => {
                video.srcObject = stream;
                video.play();
                
                const scanInterval = setInterval(() => {
                    canvas.width = video.videoWidth;
                    canvas.height = video.videoHeight;
                    context.drawImage(video, 0, 0);
                    
                    barcodeDetector.detect(canvas)
                        .then(barcodes => {
                            if (barcodes.length > 0) {
                                clearInterval(scanInterval);
                                stream.getTracks().forEach(track => track.stop());
                                callback(barcodes[0].rawValue);
                            }
                        });
                }, 100);
            })
            .catch(err => {
                console.error('Camera error:', err);
                this.showBarcodeInput(callback);
            });
    },
    
    showBarcodeInput: function(callback) {
        frappe.prompt([
            {
                fieldtype: 'Data',
                label: 'Barcode/ISBN',
                fieldname: 'barcode',
                reqd: 1
            }
        ], function(values) {
            callback(values.barcode);
        }, 'Enter Barcode', 'Scan');
    },
    
    // Book search with autocomplete
    setupBookSearch: function(field) {
        field.get_query = function() {
            return {
                query: 'library_management.api.books.search_books',
                filters: {
                    'status': 'Available'
                }
            };
        };
        
        // Add barcode scanning button
        field.$input.after(
            $('<button class="btn btn-xs btn-default" style="margin-left: 5px;">')
            .html('<i class="fa fa-barcode"></i> Scan')
            .click(() => {
                frappe.library.scanBarcode(function(barcode) {
                    // Search by ISBN/barcode
                    frappe.call({
                        method: 'library_management.api.books.get_book_by_barcode',
                        args: { barcode: barcode },
                        callback: function(r) {
                            if (r.message) {
                                field.set_value(r.message.name);
                            } else {
                                frappe.msgprint('Book not found for barcode: ' + barcode);
                            }
                        }
                    });
                });
            })
        );
    },
    
    // Format ISBN display
    formatISBN: function(isbn) {
        if (!isbn || isbn.length < 10) return isbn;
        
        if (isbn.length === 10) {
            return isbn.replace(/(\d{1})(\d{3})(\d{5})(\d{1})/, '$1-$2-$3-$4');
        } else if (isbn.length === 13) {
            return isbn.replace(/(\d{3})(\d{1})(\d{3})(\d{5})(\d{1})/, '$1-$2-$3-$4-$5');
        }
        
        return isbn;
    },
    
    // Calculate due date
    calculateDueDate: function(issueDate, loanPeriod) {
        const date = frappe.datetime.add_days(issueDate || frappe.datetime.get_today(), loanPeriod || 14);
        return date;
    },
    
    // Member quick info
    showMemberInfo: function(memberID) {
        frappe.call({
            method: 'frappe.client.get',
            args: {
                doctype: 'Library Member',
                name: memberID
            },
            callback: function(r) {
                if (r.message) {
                    const member = r.message;
                    
                    // Get current issues count
                    frappe.call({
                        method: 'frappe.client.get_count',
                        args: {
                            doctype: 'Library Issue',
                            filters: {
                                member: memberID,
                                return_date: ['is', 'not set']
                            }
                        },
                        callback: function(count_r) {
                            const currentIssues = count_r.message || 0;
                            
                            frappe.msgprint({
                                title: 'Member Information',
                                message: `
                                    <div class="member-info">
                                        <h5>${member.full_name}</h5>
                                        <p><strong>Email:</strong> ${member.email}</p>
                                        <p><strong>Member Type:</strong> ${member.membership_type}</p>
                                        <p><strong>Current Issues:</strong> ${currentIssues}</p>
                                        <p><strong>Max Books:</strong> ${member.max_books_allowed || 5}</p>
                                    </div>
                                `,
                                indicator: currentIssues === 0 ? 'green' : 'orange'
                            });
                        }
                    });
                }
            }
        });
    }
};

// Global event handlers
$(document).ready(function() {
    // Add library-specific shortcuts
    frappe.ui.keys.add_shortcut({
        shortcut: 'ctrl+shift+b',
        action: () => frappe.library.scanBarcode(console.log),
        description: 'Scan Barcode'
    });
    
    // Quick member lookup
    frappe.ui.keys.add_shortcut({
        shortcut: 'ctrl+shift+m',
        action: () => {
            frappe.prompt('Member ID', function(values) {
                frappe.library.showMemberInfo(values.member_id);
            });
        },
        description: 'Member Quick Info'
    });
});
```

### Form Customizations

```javascript
// library_management/public/js/library_book.js

frappe.ui.form.on('Library Book', {
    refresh: function(frm) {
        // Add custom buttons
        if (!frm.doc.__islocal) {
            frm.add_custom_button(__('Issue Book'), function() {
                frappe.library.issueBook(frm.doc.name);
            }, __('Actions'));
            
            frm.add_custom_button(__('View Issues'), function() {
                frappe.route_options = {
                    'book': frm.doc.name
                };
                frappe.set_route('List', 'Library Issue');
            }, __('Actions'));
            
            frm.add_custom_button(__('Print Barcode'), function() {
                frappe.library.printBarcode(frm.doc);
            }, __('Actions'));
        }
        
        // Set custom indicators
        if (frm.doc.status === 'Available') {
            frm.dashboard.set_headline_alert(__('Available for Issue'), 'green');
        } else if (frm.doc.status === 'Issued') {
            frm.dashboard.set_headline_alert(__('Currently Issued'), 'orange');
        } else if (frm.doc.status === 'Reserved') {
            frm.dashboard.set_headline_alert(__('Reserved'), 'blue');
        }
        
        // Show popularity metrics
        if (frm.doc.popularity_score) {
            frm.dashboard.add_indicator(__('Popularity: {0}', [frm.doc.popularity_score]), 'blue');
        }
    },
    
    isbn: function(frm) {
        if (frm.doc.isbn && frm.doc.isbn.length >= 10) {
            // Format ISBN display
            frm.set_value('isbn', frappe.library.formatISBN(frm.doc.isbn));
            
            // Auto-populate book details from external API
            frappe.library.populateBookDetails(frm);
        }
    },
    
    category: function(frm) {
        // Update location suggestion based on category
        if (frm.doc.category) {
            frappe.call({
                method: 'library_management.utils.get_suggested_location',
                args: {
                    category: frm.doc.category
                },
                callback: function(r) {
                    if (r.message) {
                        frm.set_value('location', r.message);
                    }
                }
            });
        }
    },
    
    validate: function(frm) {
        // Custom validation
        if (frm.doc.publication_year) {
            const currentYear = new Date().getFullYear();
            if (frm.doc.publication_year > currentYear + 1) {
                frappe.msgprint(__('Publication year cannot be in the future'));
                frappe.validated = false;
            }
        }
    }
});

// Custom library functions
frappe.library.issueBook = function(bookName) {
    frappe.prompt([
        {
            fieldtype: 'Link',
            label: 'Member',
            fieldname: 'member',
            options: 'Library Member',
            reqd: 1
        },
        {
            fieldtype: 'Date',
            label: 'Issue Date',
            fieldname: 'issue_date',
            default: frappe.datetime.get_today(),
            reqd: 1
        },
        {
            fieldtype: 'Date', 
            label: 'Due Date',
            fieldname: 'due_date',
            default: frappe.library.calculateDueDate(),
            reqd: 1
        }
    ], function(values) {
        frappe.call({
            method: 'library_management.api.issues.create_issue',
            args: {
                book: bookName,
                member: values.member,
                issue_date: values.issue_date,
                due_date: values.due_date
            },
            callback: function(r) {
                if (r.message) {
                    frappe.msgprint(__('Book issued successfully'));
                    cur_frm.reload_doc();
                }
            }
        });
    }, __('Issue Book'), __('Issue'));
};

frappe.library.populateBookDetails = function(frm) {
    frappe.call({
        method: 'library_management.integrations.books_api.get_book_details',
        args: {
            isbn: frm.doc.isbn
        },
        callback: function(r) {
            if (r.message) {
                const book_data = r.message;
                
                // Populate fields if empty
                if (!frm.doc.book_name && book_data.title) {
                    frm.set_value('book_name', book_data.title);
                }
                
                if (!frm.doc.author && book_data.authors) {
                    frm.set_value('author', book_data.authors[0]);
                }
                
                if (!frm.doc.publication_year && book_data.published_date) {
                    const year = new Date(book_data.published_date).getFullYear();
                    frm.set_value('publication_year', year);
                }
                
                if (book_data.description) {
                    frm.set_value('description', book_data.description);
                }
                
                if (book_data.cover_image_url) {
                    frm.set_value('cover_image_url', book_data.cover_image_url);
                }
            }
        }
    });
};

frappe.library.printBarcode = function(book) {
    const barcodeWindow = window.open('', '_blank');
    barcodeWindow.document.write(`
        <html>
        <head>
            <title>Barcode - ${book.book_name}</title>
            <style>
                body { font-family: Arial, sans-serif; text-align: center; margin: 20px; }
                .barcode { margin: 20px 0; }
                .book-info { margin: 10px 0; font-size: 12px; }
            </style>
        </head>
        <body>
            <h3>${book.book_name}</h3>
            <div class="book-info">Author: ${book.author}</div>
            <div class="book-info">ISBN: ${book.isbn}</div>
            <div class="barcode">
                <svg id="barcode"></svg>
            </div>
            <div class="book-info">ID: ${book.name}</div>
            <script src="https://cdn.jsdelivr.net/npm/jsbarcode@3.11.5/dist/JsBarcode.all.min.js"></script>
            <script>
                JsBarcode("#barcode", "${book.isbn || book.name}", {
                    format: "CODE128",
                    width: 2,
                    height: 50,
                    displayValue: true
                });
                setTimeout(() => window.print(), 1000);
            </script>
        </body>
        </html>
    `);
};
```

## Best Practices

### App Development Guidelines

```python
# library_management/utils/best_practices.py

class DevelopmentBestPractices:
    """Best practices for Frappe app development"""
    
    @staticmethod
    def naming_conventions():
        """Naming convention guidelines"""
        
        return {
            "apps": {
                "format": "snake_case",
                "examples": ["library_management", "hotel_booking", "school_fees"],
                "avoid": ["LibraryManagement", "library-management"]
            },
            
            "doctypes": {
                "format": "Title Case with Spaces",
                "examples": ["Library Book", "Library Member", "Book Category"],
                "avoid": ["library_book", "LibraryBook", "library-book"]
            },
            
            "fields": {
                "format": "snake_case",
                "examples": ["book_name", "issue_date", "member_id"],
                "avoid": ["bookName", "IssueDate", "book-name"]
            },
            
            "methods": {
                "format": "snake_case",
                "examples": ["get_book_details", "calculate_due_date", "send_notification"],
                "avoid": ["getBookDetails", "CalculateDueDate"]
            },
            
            "files_and_folders": {
                "format": "snake_case",
                "examples": ["library_book.py", "book_utils.py", "api/books.py"],
                "avoid": ["LibraryBook.py", "bookUtils.py"]
            }
        }
    
    @staticmethod
    def code_organization():
        """Code organization best practices"""
        
        return {
            "controllers": {
                "location": "doctype/[doctype_name]/[doctype_name].py",
                "purpose": "Business logic specific to DocType",
                "keep_light": "Delegate complex logic to utils modules"
            },
            
            "utils": {
                "location": "utils/",
                "purpose": "Reusable functions and classes",
                "examples": ["utils/member_utils.py", "utils/book_search.py"]
            },
            
            "api": {
                "location": "api/",
                "purpose": "REST API endpoints",
                "structure": "api/[resource].py (e.g., api/books.py)"
            },
            
            "integrations": {
                "location": "integrations/",
                "purpose": "Third-party system integrations",
                "examples": ["integrations/payment_gateway.py"]
            },
            
            "tasks": {
                "location": "tasks/",
                "purpose": "Background and scheduled tasks",
                "structure": "tasks/[frequency].py (e.g., tasks/daily.py)"
            }
        }
    
    @staticmethod
    def performance_optimization():
        """Performance optimization guidelines"""
        
        return {
            "database_queries": [
                "Use frappe.get_all() instead of frappe.get_doc() for lists",
                "Specify required fields explicitly",
                "Use database indexes on frequently queried fields",
                "Avoid N+1 queries in loops",
                "Use batch operations for multiple records"
            ],
            
            "caching": [
                "Cache expensive operations using @frappe.cache.cache_result",
                "Clear relevant caches when data changes",
                "Use Redis for session and temporary data",
                "Cache static data like settings and configurations"
            ],
            
            "javascript": [
                "Minimize DOM manipulations",
                "Use event delegation for dynamic content",
                "Debounce expensive operations like API calls",
                "Lazy load heavy components",
                "Use browser caching for static assets"
            ]
        }
    
    @staticmethod
    def security_guidelines():
        """Security best practices"""
        
        return {
            "input_validation": [
                "Validate all user inputs",
                "Sanitize HTML content",
                "Use parameterized queries",
                "Validate file uploads",
                "Check data types and ranges"
            ],
            
            "permissions": [
                "Use DocType permissions appropriately",
                "Implement custom permission logic when needed",
                "Validate permissions in API methods",
                "Use has_permission() in controllers",
                "Implement row-level security with User Permissions"
            ],
            
            "data_protection": [
                "Encrypt sensitive data",
                "Don't log sensitive information",
                "Use HTTPS for all communications",
                "Implement proper session management",
                "Follow GDPR/privacy regulations"
            ]
        }
    
    @staticmethod
    def testing_strategy():
        """Testing strategy recommendations"""
        
        return {
            "unit_tests": [
                "Test each method in isolation",
                "Mock external dependencies",
                "Test edge cases and error conditions",
                "Maintain high test coverage (>80%)",
                "Use descriptive test names"
            ],
            
            "integration_tests": [
                "Test complete workflows",
                "Test API endpoints",
                "Test database interactions",
                "Test email notifications",
                "Test background jobs"
            ],
            
            "test_data": [
                "Use fixtures for consistent test data",
                "Create isolated test environments",
                "Clean up test data after tests",
                "Use realistic test data",
                "Version control test data"
            ]
        }

def validate_app_structure(app_name):
    """Validate app structure against best practices"""
    
    import os
    
    app_path = frappe.get_app_path(app_name)
    issues = []
    
    # Check required files
    required_files = [
        "hooks.py",
        "modules.txt", 
        "__init__.py",
        "setup.py"
    ]
    
    for file_name in required_files:
        if not os.path.exists(os.path.join(app_path, file_name)):
            issues.append(f"Missing required file: {file_name}")
            
    # Check folder structure
    recommended_folders = [
        "fixtures",
        "patches", 
        "public/js",
        "public/css",
        "templates",
        "utils"
    ]
    
    for folder in recommended_folders:
        folder_path = os.path.join(app_path, folder)
        if not os.path.exists(folder_path):
            issues.append(f"Missing recommended folder: {folder}")
            
    return issues

def analyze_code_quality(app_name):
    """Analyze code quality metrics"""
    
    import ast
    import os
    
    app_path = frappe.get_app_path(app_name)
    quality_report = {
        "total_files": 0,
        "total_lines": 0,
        "avg_method_length": 0,
        "complex_methods": [],
        "long_files": [],
        "naming_violations": []
    }
    
    for root, dirs, files in os.walk(app_path):
        for file in files:
            if file.endswith('.py'):
                file_path = os.path.join(root, file)
                quality_report["total_files"] += 1
                
                try:
                    with open(file_path, 'r') as f:
                        content = f.read()
                        lines = content.count('\n')
                        quality_report["total_lines"] += lines
                        
                        # Check file length
                        if lines > 500:
                            quality_report["long_files"].append({
                                "file": file_path,
                                "lines": lines
                            })
                            
                        # Parse AST for method analysis
                        try:
                            tree = ast.parse(content)
                            for node in ast.walk(tree):
                                if isinstance(node, ast.FunctionDef):
                                    # Check method length
                                    method_lines = node.end_lineno - node.lineno
                                    if method_lines > 50:
                                        quality_report["complex_methods"].append({
                                            "file": file_path,
                                            "method": node.name,
                                            "lines": method_lines
                                        })
                                        
                        except SyntaxError:
                            pass  # Skip files with syntax errors
                            
                except Exception:
                    pass  # Skip files that can't be read
                    
    return quality_report
```

---

**Next Steps**: Continue with [Deployment & Production](10-deployment-production.md) to learn about production deployment strategies, performance optimization, monitoring, and scaling Frappe applications.