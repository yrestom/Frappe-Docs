# Server-Side Development - Complete Guide

> **Comprehensive reference for Python backend development in Frappe Framework**

## Table of Contents

- [Python Controller Development](#python-controller-development)
- [Document ORM & Database Operations](#document-orm--database-operations)
- [API Development & Whitelisting](#api-development--whitelisting)
- [Background Jobs & Scheduling](#background-jobs--scheduling)
- [Email System Integration](#email-system-integration)
- [Caching & Performance](#caching--performance)
- [Error Handling & Logging](#error-handling--logging)
- [File Management](#file-management)
- [Utility Functions & Helpers](#utility-functions--helpers)
- [Advanced Patterns](#advanced-patterns)

## Python Controller Development

### Document Controller Structure

**Basic Controller Pattern:**
```python
# apps/library_management/library_management/doctype/library_member/library_member.py
import frappe
from frappe import _, throw, msgprint
from frappe.model.document import Document
from frappe.utils import nowdate, date_diff, cint, flt, get_datetime

class LibraryMember(Document):
    """Controller for Library Member DocType"""
    
    def validate(self):
        """Primary validation method - called on every save"""
        self.validate_email_uniqueness()
        self.validate_membership_eligibility()
        self.calculate_membership_fee()
    
    def before_save(self):
        """Pre-save processing"""
        self.set_full_name()
        self.standardize_data()
    
    def after_insert(self):
        """Post-creation actions"""
        self.create_user_account()
        self.send_welcome_email()
        self.update_membership_statistics()
    
    def on_update(self):
        """Called after every save"""
        self.sync_user_account_data()
    
    def on_submit(self):
        """Called when document is submitted"""
        if not self.membership_card_generated:
            self.generate_membership_card()
    
    def on_cancel(self):
        """Called when document is cancelled"""
        self.deactivate_related_services()
    
    def validate_email_uniqueness(self):
        """Custom validation method"""
        if self.email:
            existing = frappe.db.exists("Library Member", {
                "email": self.email,
                "name": ("!=", self.name)
            })
            if existing:
                frappe.throw(_("Member with email {0} already exists").format(self.email))
```

### Lifecycle Hook Patterns

**Complete Hook Implementation:**
```python
class LibraryBook(Document):
    def __init__(self, *args, **kwargs):
        """Constructor - rare to override"""
        super().__init__(*args, **kwargs)
        self.setup_default_values()
    
    def before_validate(self):
        """Called before validate() - data cleanup"""
        self.isbn = self.isbn.replace("-", "") if self.isbn else None
        self.title = self.title.strip() if self.title else None
    
    def validate(self):
        """Main validation - business rules"""
        self.validate_isbn_format()
        self.validate_publication_date()
        self.check_duplicate_books()
        self.calculate_book_value()
    
    def after_validate(self):
        """Post-validation processing"""
        self.set_book_category()
        self.update_search_keywords()
    
    def before_insert(self):
        """Pre-creation setup"""
        self.set_book_id()
        self.set_acquisition_details()
    
    def after_insert(self):
        """Post-creation actions"""
        self.create_barcode()
        self.update_library_catalog()
        self.notify_librarians()
    
    def before_save(self):
        """Pre-save (insert/update)"""
        self.last_modified_by = frappe.session.user
        self.validate_changes()
    
    def after_save(self):
        """Post-save actions"""
        self.clear_related_cache()
        self.update_statistics()
    
    def on_update_after_submit(self):
        """Special hook for submitted documents"""
        # Only certain fields can be updated after submission
        allowed_fields = ['location', 'condition', 'notes']
        for field in allowed_fields:
            if self.has_value_changed(field):
                self.log_change(field)
    
    def before_rename(self, old_name, new_name, merge=False):
        """Pre-rename validations"""
        if frappe.db.exists("Book Issue", {"book": old_name, "status": "Issued"}):
            frappe.throw(_("Cannot rename book that is currently issued"))
    
    def after_rename(self, old_name, new_name, merge=False):
        """Post-rename cleanup"""
        # Update references in related documents
        frappe.db.set_value("Book Location", {"book": old_name}, "book", new_name)
    
    def on_trash(self):
        """Pre-deletion validations"""
        if frappe.db.exists("Book Issue", {"book": self.name}):
            frappe.throw(_("Cannot delete book with issue history"))
    
    def after_delete(self):
        """Post-deletion cleanup"""
        # Clean up related records
        frappe.db.delete("Book Barcode", {"book": self.name})
```

### Document State Management

**Status and Workflow Control:**
```python
class BookIssue(Document):
    def validate(self):
        """Status-based validation"""
        if self.status == "Issued":
            self.validate_issue_rules()
        elif self.status == "Returned":
            self.validate_return_rules()
        elif self.status == "Overdue":
            self.calculate_fine_amount()
    
    def validate_issue_rules(self):
        """Issue-specific validations"""
        # Check member eligibility
        member = frappe.get_doc("Library Member", self.member)
        if not member.is_active:
            frappe.throw(_("Cannot issue book to inactive member"))
        
        # Check book availability
        if not self.is_book_available():
            frappe.throw(_("Book is not available for issue"))
        
        # Check member's book limit
        current_issues = frappe.db.count("Book Issue", {
            "member": self.member,
            "status": "Issued"
        })
        
        max_books = frappe.get_value("Membership Type", 
            member.membership_type, "max_books_allowed")
        
        if current_issues >= max_books:
            frappe.throw(_("Member has reached maximum book limit"))
    
    def on_submit(self):
        """Issue the book"""
        if self.status == "Issued":
            # Update book status
            frappe.db.set_value("Library Book", self.book, "status", "Issued")
            
            # Set expected return date
            self.expected_return_date = frappe.utils.add_days(
                self.issue_date, self.get_loan_duration()
            )
            
            # Send notification
            self.send_issue_confirmation()
    
    def return_book(self, return_date=None):
        """Custom method for returning books"""
        return_date = return_date or frappe.utils.nowdate()
        
        # Calculate fine if overdue
        if frappe.utils.getdate(return_date) > frappe.utils.getdate(self.expected_return_date):
            self.calculate_fine(return_date)
        
        # Update status
        self.status = "Returned"
        self.actual_return_date = return_date
        
        # Update book availability
        frappe.db.set_value("Library Book", self.book, "status", "Available")
        
        # Save changes
        self.save()
        
        return {"status": "success", "fine_amount": self.fine_amount or 0}
```

## Document ORM & Database Operations

### Database Query Patterns

**Using frappe.get_doc() variants:**
```python
# Get single document
member = frappe.get_doc("Library Member", "LIB-001")

# Get single with specific fields
member = frappe.get_doc("Library Member", "LIB-001")
name_email = [member.member_name, member.email]

# Get for update (with row lock)
member = frappe.get_doc("Library Member", "LIB-001", for_update=True)

# Get cached document (from cache if available)
member = frappe.get_cached_doc("Library Member", "LIB-001")
```

**Using frappe.get_all() for lists:**
```python
# Basic list query
active_members = frappe.get_all("Library Member",
    filters={"status": "Active"},
    fields=["name", "member_name", "email"]
)

# Complex filters
overdue_issues = frappe.get_all("Book Issue",
    filters={
        "status": "Issued",
        "expected_return_date": ["<", frappe.utils.nowdate()],
        "member": ["like", "LIB-%"]
    },
    fields=["name", "member", "book", "expected_return_date"],
    order_by="expected_return_date asc",
    limit_page_length=50
)

# Using OR filters
members = frappe.get_all("Library Member",
    filters={
        "status": "Active"
    },
    or_filters={
        "membership_type": "Premium",
        "total_books_issued": [">", 10]
    }
)
```

### Raw SQL Operations

**Direct SQL queries:**
```python
# Simple query with parameters
results = frappe.db.sql("""
    SELECT name, member_name, email 
    FROM `tabLibrary Member` 
    WHERE status = %s AND membership_type = %s
""", ("Active", "Premium"), as_dict=True)

# Complex analytical query
monthly_stats = frappe.db.sql("""
    SELECT 
        MONTH(issue_date) as month,
        YEAR(issue_date) as year,
        COUNT(*) as issues_count,
        COUNT(DISTINCT member) as unique_members
    FROM `tabBook Issue`
    WHERE issue_date >= %s
    GROUP BY YEAR(issue_date), MONTH(issue_date)
    ORDER BY year, month
""", (frappe.utils.add_months(frappe.utils.nowdate(), -12),), as_dict=True)

# Using SQL with proper escaping
def get_member_statistics(member_id):
    return frappe.db.sql("""
        SELECT 
            COUNT(*) as total_issues,
            SUM(CASE WHEN status = 'Returned' THEN 1 ELSE 0 END) as returned,
            SUM(CASE WHEN status = 'Overdue' THEN 1 ELSE 0 END) as overdue,
            AVG(DATEDIFF(actual_return_date, issue_date)) as avg_loan_days
        FROM `tabBook Issue`
        WHERE member = %(member)s
    """, {"member": member_id}, as_dict=True)[0]
```

### Database Utilities

**Quick database operations:**
```python
# Get single value
member_name = frappe.db.get_value("Library Member", "LIB-001", "member_name")

# Get multiple values
member_details = frappe.db.get_value("Library Member", "LIB-001", 
    ["member_name", "email", "membership_type"], as_dict=True)

# Set single value (bypasses validation)
frappe.db.set_value("Library Member", "LIB-001", "last_visit", frappe.utils.now())

# Check existence
if frappe.db.exists("Library Member", {"email": "john@example.com"}):
    frappe.throw(_("Member already exists"))

# Count records
active_count = frappe.db.count("Library Member", {"status": "Active"})

# Delete records
frappe.db.delete("Book Issue", {"status": "Cancelled"})

# Database transactions
def transfer_books_between_branches(from_branch, to_branch, book_list):
    try:
        frappe.db.begin()
        
        for book in book_list:
            # Update book location
            frappe.db.set_value("Library Book", book, "branch", to_branch)
            
            # Log transfer
            transfer_log = frappe.get_doc({
                "doctype": "Book Transfer Log",
                "book": book,
                "from_branch": from_branch,
                "to_branch": to_branch,
                "transfer_date": frappe.utils.nowdate()
            })
            transfer_log.insert()
        
        frappe.db.commit()
        return {"success": True, "transferred_count": len(book_list)}
        
    except Exception as e:
        frappe.db.rollback()
        frappe.throw(_("Transfer failed: {0}").format(str(e)))
```

### Query Builder Usage

```python
# Using Frappe Query Builder (PyPika-based)
from frappe.query_builder import DocType

# Basic query builder usage
Member = DocType("Library Member")
Issue = DocType("Book Issue")

# Simple select
active_members = (
    frappe.qb.from_(Member)
    .select(Member.name, Member.member_name, Member.email)
    .where(Member.status == "Active")
    .where(Member.membership_type.isin(["Premium", "Standard"]))
    .orderby(Member.member_name)
    .limit(50)
).run(as_dict=True)

# Join queries
member_issues = (
    frappe.qb.from_(Member)
    .join(Issue).on(Member.name == Issue.member)
    .select(
        Member.name,
        Member.member_name,
        Issue.book,
        Issue.issue_date,
        Issue.status
    )
    .where(Member.status == "Active")
    .where(Issue.status == "Issued")
).run(as_dict=True)

# Aggregation
issue_stats = (
    frappe.qb.from_(Issue)
    .select(
        Issue.member,
        frappe.qb.functions.Count("*").as_("total_issues"),
        frappe.qb.functions.Max(Issue.issue_date).as_("last_issue")
    )
    .groupby(Issue.member)
    .having(frappe.qb.functions.Count("*") > 5)
).run(as_dict=True)
```

## API Development & Whitelisting

### Creating API Endpoints

**Basic API Method Pattern:**
```python
# apps/library_management/library_management/api.py
import frappe
from frappe import _

@frappe.whitelist()
def get_member_dashboard(member):
    """Get dashboard data for a library member"""
    
    # Verify permissions
    if not frappe.has_permission("Library Member", "read", member):
        frappe.throw(_("Not permitted"), frappe.PermissionError)
    
    member_doc = frappe.get_doc("Library Member", member)
    
    # Current issues
    current_issues = frappe.get_all("Book Issue",
        filters={"member": member, "status": "Issued"},
        fields=["name", "book", "issue_date", "expected_return_date"]
    )
    
    # Issue history
    issue_history = frappe.get_all("Book Issue",
        filters={"member": member},
        fields=["book", "issue_date", "actual_return_date", "status"],
        order_by="issue_date desc",
        limit_page_length=10
    )
    
    # Overdue books
    overdue_books = frappe.get_all("Book Issue",
        filters={
            "member": member,
            "status": "Issued",
            "expected_return_date": ["<", frappe.utils.nowdate()]
        },
        fields=["book", "expected_return_date"]
    )
    
    return {
        "member_info": {
            "name": member_doc.member_name,
            "email": member_doc.email,
            "membership_type": member_doc.membership_type
        },
        "current_issues": current_issues,
        "issue_history": issue_history,
        "overdue_books": overdue_books,
        "statistics": {
            "total_books_issued": len(issue_history),
            "current_issues_count": len(current_issues),
            "overdue_count": len(overdue_books)
        }
    }

@frappe.whitelist()
def issue_book(member, book, issue_date=None):
    """Issue a book to a member"""
    
    # Validate permissions
    if not frappe.has_permission("Book Issue", "create"):
        frappe.throw(_("Not permitted"), frappe.PermissionError)
    
    # Create book issue document
    issue_doc = frappe.get_doc({
        "doctype": "Book Issue",
        "member": member,
        "book": book,
        "issue_date": issue_date or frappe.utils.nowdate(),
        "status": "Issued"
    })
    
    # Validate and save
    issue_doc.insert()
    issue_doc.submit()
    
    return {
        "success": True,
        "issue_id": issue_doc.name,
        "expected_return_date": issue_doc.expected_return_date
    }

@frappe.whitelist(allow_guest=True)
def search_books(query, limit=20):
    """Public API to search books - guest access allowed"""
    
    # Sanitize input
    query = frappe.utils.sanitize_html(query.strip())
    
    if len(query) < 3:
        frappe.throw(_("Search query must be at least 3 characters"))
    
    books = frappe.get_all("Library Book",
        filters={
            "status": "Available"
        },
        or_filters={
            "title": ["like", f"%{query}%"],
            "author": ["like", f"%{query}%"],
            "isbn": ["like", f"%{query}%"]
        },
        fields=["name", "title", "author", "isbn", "publication_year"],
        limit_page_length=limit
    )
    
    return books
```

### Advanced API Patterns

**File Upload API:**
```python
@frappe.whitelist()
def upload_member_photo(member):
    """Upload member photo"""
    
    if not frappe.has_permission("Library Member", "write", member):
        frappe.throw(_("Not permitted"), frappe.PermissionError)
    
    # Handle file upload
    from frappe.utils.file_manager import save_file
    
    if not frappe.request.files:
        frappe.throw(_("No file uploaded"))
    
    file_obj = frappe.request.files['file']
    
    # Validate file type
    allowed_types = ['image/jpeg', 'image/png', 'image/gif']
    if file_obj.content_type not in allowed_types:
        frappe.throw(_("Only image files are allowed"))
    
    # Save file
    file_doc = save_file(
        file_obj.filename,
        file_obj.read(),
        "Library Member",
        member,
        is_private=0
    )
    
    # Update member document
    frappe.db.set_value("Library Member", member, "photo", file_doc.file_url)
    
    return {
        "success": True,
        "file_url": file_doc.file_url,
        "file_name": file_doc.file_name
    }

@frappe.whitelist(methods=["POST"])
def bulk_update_members(members_data):
    """Bulk update multiple members"""
    
    if not frappe.has_permission("Library Member", "write"):
        frappe.throw(_("Not permitted"), frappe.PermissionError)
    
    updated_members = []
    errors = []
    
    for member_data in members_data:
        try:
            member_id = member_data.get("name")
            if not member_id:
                continue
            
            # Get and update member
            member_doc = frappe.get_doc("Library Member", member_id)
            
            # Update allowed fields only
            allowed_fields = ["phone", "address", "emergency_contact"]
            for field in allowed_fields:
                if field in member_data:
                    setattr(member_doc, field, member_data[field])
            
            member_doc.save()
            updated_members.append(member_id)
            
        except Exception as e:
            errors.append({
                "member": member_data.get("name", "Unknown"),
                "error": str(e)
            })
    
    return {
        "success": True,
        "updated_count": len(updated_members),
        "updated_members": updated_members,
        "errors": errors
    }
```

### Rate Limiting and Security

```python
@frappe.whitelist()
@frappe.rate_limit(limit=10, window=60)  # 10 requests per minute
def search_members_advanced(search_term, filters=None):
    """Rate-limited member search"""
    
    # Input validation
    if not search_term or len(search_term) < 2:
        frappe.throw(_("Search term must be at least 2 characters"))
    
    # Sanitize input
    search_term = frappe.utils.sanitize_html(search_term)
    
    # Permission check
    if not frappe.has_permission("Library Member", "read"):
        frappe.throw(_("Not permitted"), frappe.PermissionError)
    
    # Build query
    query_filters = {"status": "Active"}
    
    if filters:
        # Validate allowed filters
        allowed_filters = ["membership_type", "branch", "status"]
        for key, value in filters.items():
            if key in allowed_filters:
                query_filters[key] = value
    
    # Search
    members = frappe.get_all("Library Member",
        filters=query_filters,
        or_filters={
            "member_name": ["like", f"%{search_term}%"],
            "email": ["like", f"%{search_term}%"]
        },
        fields=["name", "member_name", "email", "membership_type"],
        limit_page_length=50
    )
    
    return members
```

## Background Jobs & Scheduling

### Background Job System

**Basic Job Enqueuing:**
```python
# From frappe/utils/background_jobs.py analysis
import frappe

# Simple background job
@frappe.whitelist()
def generate_monthly_report(month, year):
    """Generate monthly library report"""
    
    # Enqueue the job
    frappe.enqueue(
        method="library_management.reports.generate_monthly_report",
        queue="long",  # Use long queue for heavy operations
        timeout=1500,   # 25 minutes timeout
        month=month,
        year=year,
        user=frappe.session.user  # Pass current user context
    )
    
    return {"success": True, "message": "Report generation started"}

# The actual background method
def generate_monthly_report(month, year, user=None):
    """Background method to generate report"""
    
    # Set user context in background job
    if user:
        frappe.set_user(user)
    
    try:
        # Generate report data
        report_data = compile_monthly_statistics(month, year)
        
        # Create report document
        report_doc = frappe.get_doc({
            "doctype": "Library Report",
            "report_type": "Monthly Statistics",
            "month": month,
            "year": year,
            "data": frappe.as_json(report_data),
            "generated_by": user or "System"
        })
        report_doc.insert()
        
        # Send email notification
        if user:
            send_report_completion_email(user, report_doc.name)
        
        frappe.db.commit()
        return {"success": True, "report_id": report_doc.name}
        
    except Exception as e:
        frappe.db.rollback()
        frappe.log_error(f"Monthly report generation failed: {str(e)}")
        raise
```

**Job with Progress Tracking:**
```python
def bulk_import_books(file_path, user=None):
    """Import books from CSV file with progress tracking"""
    
    if user:
        frappe.set_user(user)
    
    import csv
    from frappe.utils.csv import read_csv_content
    
    # Read CSV file
    content = read_csv_content(file_path)
    total_rows = len(content)
    
    success_count = 0
    error_count = 0
    errors = []
    
    for i, row in enumerate(content):
        try:
            # Update progress
            progress = (i + 1) / total_rows * 100
            frappe.publish_progress(
                percent=progress,
                title="Importing Books",
                description=f"Processing row {i+1} of {total_rows}"
            )
            
            # Create book document
            book_doc = frappe.get_doc({
                "doctype": "Library Book",
                "title": row.get("title"),
                "author": row.get("author"),
                "isbn": row.get("isbn"),
                "publication_year": row.get("year"),
                "category": row.get("category", "General")
            })
            
            book_doc.insert()
            success_count += 1
            
        except Exception as e:
            error_count += 1
            errors.append({
                "row": i + 1,
                "data": row,
                "error": str(e)
            })
            
            # Continue with next row
            continue
    
    # Final result
    result = {
        "total_processed": total_rows,
        "success_count": success_count,
        "error_count": error_count,
        "errors": errors[:50]  # Limit error details
    }
    
    # Save import log
    import_log = frappe.get_doc({
        "doctype": "Import Log",
        "import_type": "Book Import",
        "file_path": file_path,
        "result": frappe.as_json(result),
        "imported_by": user
    })
    import_log.insert()
    
    frappe.db.commit()
    return result
```

### Scheduled Jobs

**Cron-style Scheduling:**
```python
# hooks.py - Define scheduled jobs
scheduler_events = {
    "cron": {
        # Every day at 2 AM
        "0 2 * * *": [
            "library_management.tasks.daily_overdue_check"
        ],
        # Every Monday at 8 AM  
        "0 8 * * 1": [
            "library_management.tasks.weekly_inventory_check"
        ],
        # First day of every month at 1 AM
        "0 1 1 * *": [
            "library_management.tasks.monthly_statistics"
        ]
    },
    "daily": [
        "library_management.tasks.cleanup_expired_reservations",
        "library_management.tasks.send_return_reminders"
    ],
    "weekly": [
        "library_management.tasks.generate_weekly_report"
    ],
    "monthly": [
        "library_management.tasks.archive_old_records"
    ]
}

# Task implementation
def daily_overdue_check():
    """Check for overdue books and notify members"""
    
    overdue_issues = frappe.get_all("Book Issue",
        filters={
            "status": "Issued",
            "expected_return_date": ["<", frappe.utils.nowdate()]
        },
        fields=["name", "member", "book", "expected_return_date"]
    )
    
    for issue in overdue_issues:
        # Update status to overdue
        frappe.db.set_value("Book Issue", issue.name, "status", "Overdue")
        
        # Calculate and set fine
        days_overdue = frappe.utils.date_diff(
            frappe.utils.nowdate(), 
            issue.expected_return_date
        )
        fine_amount = days_overdue * 2  # $2 per day
        
        frappe.db.set_value("Book Issue", issue.name, "fine_amount", fine_amount)
        
        # Send notification
        send_overdue_notification(issue.member, issue.book, days_overdue, fine_amount)
    
    frappe.db.commit()
    return f"Processed {len(overdue_issues)} overdue issues"
```

### Task Decorator Pattern

```python
# Using the @frappe.task decorator
@frappe.task(queue='long', timeout=3600)
def generate_annual_report(year):
    """Generate comprehensive annual report"""
    
    # This method can be called directly or enqueued
    report_data = {
        "year": year,
        "total_members": frappe.db.count("Library Member"),
        "total_books": frappe.db.count("Library Book"),
        "total_issues": frappe.db.count("Book Issue", {"YEAR(issue_date)": year}),
        "popular_books": get_popular_books(year),
        "member_statistics": get_member_statistics(year)
    }
    
    return report_data

# Usage examples:
# Synchronous execution
result = generate_annual_report(2024)

# Asynchronous execution  
job = generate_annual_report.enqueue(year=2024, queue="long")
```

## Email System Integration

### Email Sending Patterns

**Basic Email Sending:**
```python
def send_welcome_email(member_doc):
    """Send welcome email to new member"""
    
    # Using frappe.sendmail
    frappe.sendmail(
        recipients=[member_doc.email],
        subject=_("Welcome to Our Library!"),
        template="member_welcome",  # Email template name
        args={
            "member_name": member_doc.member_name,
            "membership_type": member_doc.membership_type,
            "member_id": member_doc.name,
            "library_url": frappe.utils.get_url()
        },
        header=["Welcome to Library", "green"],
        delayed=True,  # Send via background queue
        reference_doctype="Library Member",
        reference_name=member_doc.name
    )

def send_overdue_notification(member, book, days_overdue, fine_amount):
    """Send overdue book notification"""
    
    member_doc = frappe.get_doc("Library Member", member)
    book_doc = frappe.get_doc("Library Book", book)
    
    # Custom email content
    message = f"""
    <h3>Overdue Book Notification</h3>
    <p>Dear {member_doc.member_name},</p>
    <p>Your book "{book_doc.title}" is overdue by {days_overdue} days.</p>
    <p>Fine amount: ${fine_amount}</p>
    <p>Please return the book as soon as possible.</p>
    """
    
    frappe.sendmail(
        recipients=[member_doc.email],
        subject=f"Overdue Book: {book_doc.title}",
        message=message,
        delayed=True,
        send_priority=2  # Higher priority for notifications
    )
```

**Email Templates:**
```html
<!-- apps/library_management/library_management/templates/emails/member_welcome.html -->
<div style="font-family: Arial, sans-serif;">
    <h2 style="color: #2e8b57;">Welcome to {{ library_name }}!</h2>
    
    <p>Dear {{ member_name }},</p>
    
    <p>Welcome to our library! Your membership has been activated successfully.</p>
    
    <div style="background: #f5f5f5; padding: 15px; margin: 20px 0;">
        <strong>Membership Details:</strong><br>
        Member ID: {{ member_id }}<br>
        Membership Type: {{ membership_type }}<br>
        Maximum Books: {{ max_books_allowed }}
    </div>
    
    <p>You can now:</p>
    <ul>
        <li>Browse and reserve books online</li>
        <li>View your reading history</li>
        <li>Receive notifications about due dates</li>
    </ul>
    
    <p><a href="{{ library_url }}" style="background: #2e8b57; color: white; padding: 10px 20px; text-decoration: none;">Visit Library Portal</a></p>
    
    <p>Happy Reading!<br>
    The Library Team</p>
</div>
```

**Bulk Email Operations:**
```python
def send_monthly_newsletter(member_list=None):
    """Send monthly newsletter to members"""
    
    if not member_list:
        # Get all active members
        member_list = frappe.get_all("Library Member",
            filters={"status": "Active", "email": ["!=", ""]},
            fields=["name", "member_name", "email", "membership_type"]
        )
    
    # Get newsletter content
    newsletter = frappe.get_single("Library Newsletter Settings")
    
    # Batch send emails
    batch_size = 50
    for i in range(0, len(member_list), batch_size):
        batch = member_list[i:i + batch_size]
        
        # Enqueue batch
        frappe.enqueue(
            method="library_management.email.send_newsletter_batch",
            queue="default",
            batch=batch,
            newsletter_content=newsletter.content,
            subject=newsletter.subject
        )
    
    return f"Newsletter queued for {len(member_list)} members"

def send_newsletter_batch(batch, newsletter_content, subject):
    """Send newsletter to a batch of members"""
    
    for member in batch:
        try:
            frappe.sendmail(
                recipients=[member.email],
                subject=subject,
                message=newsletter_content.format(
                    member_name=member.member_name,
                    member_id=member.name
                ),
                delayed=False,  # Send immediately in background
                reference_doctype="Library Member",
                reference_name=member.name
            )
        except Exception as e:
            frappe.log_error(f"Newsletter send failed for {member.email}: {str(e)}")
```

## Caching & Performance

### Caching Strategies

**Request-Level Caching:**
```python
# Using frappe.local_cache
def get_popular_books(category=None):
    """Get popular books with request-level caching"""
    
    cache_key = f"popular_books_{category or 'all'}"
    
    return frappe.local_cache(
        "popular_books",
        cache_key,
        lambda: _fetch_popular_books(category),
        regenerate_if_none=True
    )

def _fetch_popular_books(category):
    """Expensive operation to fetch popular books"""
    
    filters = {}
    if category:
        filters["category"] = category
    
    popular = frappe.db.sql("""
        SELECT 
            b.name, b.title, b.author,
            COUNT(bi.name) as issue_count
        FROM `tabLibrary Book` b
        LEFT JOIN `tabBook Issue` bi ON b.name = bi.book
        WHERE b.status = 'Available'
        {category_filter}
        GROUP BY b.name
        ORDER BY issue_count DESC, b.title
        LIMIT 10
    """.format(
        category_filter="AND b.category = %(category)s" if category else ""
    ), {"category": category}, as_dict=True)
    
    return popular
```

**Redis Caching:**
```python
def get_member_statistics(member):
    """Get member statistics with Redis caching"""
    
    cache_key = f"member_stats:{member}"
    
    # Try to get from cache
    cached_stats = frappe.cache.get_value(cache_key)
    if cached_stats:
        return cached_stats
    
    # Calculate statistics
    stats = frappe.db.sql("""
        SELECT 
            COUNT(*) as total_issues,
            COUNT(CASE WHEN status = 'Returned' THEN 1 END) as returned_count,
            COUNT(CASE WHEN status = 'Overdue' THEN 1 END) as overdue_count,
            AVG(DATEDIFF(COALESCE(actual_return_date, CURDATE()), issue_date)) as avg_loan_days
        FROM `tabBook Issue`
        WHERE member = %(member)s
    """, {"member": member}, as_dict=True)[0]
    
    # Cache for 1 hour
    frappe.cache.set_value(cache_key, stats, expires_in_sec=3600)
    
    return stats

def clear_member_cache(member):
    """Clear member-related cache"""
    frappe.cache.delete_value(f"member_stats:{member}")
```

**Database Optimization:**
```python
def get_overdue_books_optimized():
    """Optimized query for overdue books"""
    
    # Use indexes and avoid N+1 queries
    overdue_data = frappe.db.sql("""
        SELECT 
            bi.name as issue_id,
            bi.member,
            bi.book,
            bi.expected_return_date,
            bi.fine_amount,
            lm.member_name,
            lm.email,
            lb.title as book_title,
            lb.author
        FROM `tabBook Issue` bi
        INNER JOIN `tabLibrary Member` lm ON bi.member = lm.name
        INNER JOIN `tabLibrary Book` lb ON bi.book = lb.name
        WHERE bi.status = 'Issued' 
        AND bi.expected_return_date < CURDATE()
        ORDER BY bi.expected_return_date ASC
    """, as_dict=True)
    
    return overdue_data

# Batch operations for performance
def bulk_update_book_status(book_ids, new_status):
    """Bulk update book status"""
    
    # Use raw SQL for bulk operations
    frappe.db.sql("""
        UPDATE `tabLibrary Book` 
        SET status = %(status)s,
            modified = NOW(),
            modified_by = %(user)s
        WHERE name IN ({placeholders})
    """.format(placeholders=', '.join(['%s'] * len(book_ids))),
    [new_status, frappe.session.user] + book_ids)
    
    # Clear relevant caches
    for book_id in book_ids:
        frappe.clear_cache(doctype="Library Book", name=book_id)
```

## Error Handling & Logging

### Exception Handling Patterns

**Structured Error Handling:**
```python
def process_book_return(issue_id, return_date=None):
    """Process book return with comprehensive error handling"""
    
    try:
        # Validate inputs
        if not issue_id:
            frappe.throw(_("Issue ID is required"))
        
        if not frappe.db.exists("Book Issue", issue_id):
            frappe.throw(_("Book issue not found"))
        
        # Get issue document
        issue_doc = frappe.get_doc("Book Issue", issue_id)
        
        # Business logic validation
        if issue_doc.status != "Issued":
            frappe.throw(_("Book is not currently issued"))
        
        # Process return
        return_date = return_date or frappe.utils.nowdate()
        
        # Calculate fine if overdue
        fine_amount = 0
        if frappe.utils.getdate(return_date) > frappe.utils.getdate(issue_doc.expected_return_date):
            days_overdue = frappe.utils.date_diff(return_date, issue_doc.expected_return_date)
            fine_amount = days_overdue * 2  # $2 per day
        
        # Update issue
        issue_doc.status = "Returned"
        issue_doc.actual_return_date = return_date
        issue_doc.fine_amount = fine_amount
        issue_doc.save()
        
        # Update book availability
        frappe.db.set_value("Library Book", issue_doc.book, "status", "Available")
        
        frappe.db.commit()
        
        return {
            "success": True,
            "fine_amount": fine_amount,
            "return_date": return_date
        }
        
    except frappe.ValidationError:
        # Re-raise validation errors as-is
        raise
        
    except frappe.PermissionError:
        # Re-raise permission errors
        raise
        
    except Exception as e:
        # Log unexpected errors
        frappe.log_error(
            message=f"Book return processing failed for issue {issue_id}: {str(e)}",
            title="Book Return Error"
        )
        
        # Rollback any changes
        frappe.db.rollback()
        
        # Throw user-friendly error
        frappe.throw(_("An error occurred while processing the book return. Please try again."))
```

### Logging Strategies

**Structured Logging:**
```python
import frappe
import json

def log_member_activity(member, activity, details=None):
    """Log member activities for audit trail"""
    
    log_data = {
        "member": member,
        "activity": activity,
        "timestamp": frappe.utils.now_datetime(),
        "user": frappe.session.user,
        "ip_address": frappe.local.request_ip,
        "details": details or {}
    }
    
    # Use structured logging
    frappe.logger("member_activity").info(json.dumps(log_data))
    
    # Also store in database for reporting
    activity_log = frappe.get_doc({
        "doctype": "Member Activity Log",
        "member": member,
        "activity": activity,
        "details": frappe.as_json(details),
        "activity_date": frappe.utils.nowdate()
    })
    activity_log.insert(ignore_permissions=True)

def log_system_performance(operation, duration, details=None):
    """Log performance metrics"""
    
    if duration > 5:  # Log slow operations (> 5 seconds)
        perf_data = {
            "operation": operation,
            "duration_seconds": duration,
            "details": details,
            "timestamp": frappe.utils.now_datetime()
        }
        
        frappe.logger("performance").warning(f"Slow operation detected: {json.dumps(perf_data)}")

# Usage with context manager
from contextlib import contextmanager
import time

@contextmanager
def performance_monitor(operation_name):
    """Context manager for performance monitoring"""
    start_time = time.time()
    try:
        yield
    finally:
        duration = time.time() - start_time
        log_system_performance(operation_name, duration)

# Usage example:
def bulk_process_returns(return_data):
    with performance_monitor("bulk_process_returns"):
        # Process returns
        for item in return_data:
            process_book_return(item['issue_id'], item['return_date'])
```

---

**Next Steps**: Continue with [Client-Side Development](04-client-side-development.md) to learn JavaScript patterns and UI customization techniques.