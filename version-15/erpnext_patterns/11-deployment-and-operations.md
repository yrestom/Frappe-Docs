# ERPNext Deployment and Operations Patterns

## Table of Contents

1. [Overview](#overview)
2. [Deployment Activation System](#deployment-activation-system)
3. [Background Job Processing](#background-job-processing)
4. [Scheduled Operations Framework](#scheduled-operations-framework)
5. [Bulk Operations Management](#bulk-operations-management)
6. [System Health Monitoring](#system-health-monitoring)
7. [Data Migration Patterns](#data-migration-patterns)
8. [Performance Optimization](#performance-optimization)
9. [Backup and Recovery](#backup-and-recovery)
10. [Production Deployment Strategies](#production-deployment-strategies)
11. [Monitoring and Alerting](#monitoring-and-alerting)
12. [Troubleshooting Patterns](#troubleshooting-patterns)
13. [Best Practices](#best-practices)

## Overview

ERPNext implements sophisticated deployment and operational patterns that ensure reliable, scalable, and maintainable production systems. This analysis examines proven patterns from ERPNext's operational infrastructure to provide comprehensive guidance for enterprise deployment and management.

### Core Operational Principles

1. **Activation-Based Deployment**: Measure system adoption and guide users through setup
2. **Background Processing**: Handle resource-intensive operations asynchronously
3. **Scheduled Automation**: Comprehensive cron-based task management
4. **Error Resilience**: Robust error handling with retry mechanisms
5. **Health Monitoring**: Continuous system health assessment
6. **Scalable Architecture**: Patterns that support enterprise-scale operations

## Deployment Activation System

ERPNext implements a sophisticated activation system to measure deployment success and guide users through system setup.

### Activation Level Measurement
*Reference: `/home/frappe/frappe-bench/apps/erpnext/erpnext/utilities/activation.py:12-63`*

```python
def get_level():
    activation_level = 0
    sales_data = []
    doctypes = {
        "Asset": 5,
        "BOM": 3,
        "Customer": 5,
        "Delivery Note": 5,
        "Employee": 3,
        "Issue": 5,
        "Item": 5,
        "Journal Entry": 3,
        "Lead": 3,
        "Material Request": 5,
        "Opportunity": 5,
        "Payment Entry": 2,
        "Project": 5,
        "Purchase Order": 2,
        "Purchase Invoice": 5,
        "Purchase Receipt": 5,
        "Quotation": 3,
        "Sales Order": 2,
        "Sales Invoice": 2,
        "Stock Entry": 3,
        "Supplier": 5,
        "Task": 5,
        "User": 5,
        "Work Order": 5,
    }

    for doctype, min_count in doctypes.items():
        count = frappe.db.count(doctype)
        if count > min_count:
            activation_level += 1
        sales_data.append({doctype: count})

    # Check setup wizard completion
    if "erpnext" in get_setup_wizard_completed_apps():
        activation_level += 1

    # Check communication activity
    communication_number = frappe.db.count("Communication", dict(communication_medium="Email"))
    if communication_number > 10:
        activation_level += 1
    
    # Check recent activity
    if frappe.db.sql("select name from tabUser where last_login > date_sub(now(), interval 2 day) limit 1"):
        activation_level += 1

    return {"activation_level": activation_level, "sales_data": sales_data}
```

### Deployment Health Assessment

```python
def assess_deployment_health():
    """Comprehensive deployment health assessment"""
    health_metrics = {
        "activation_level": get_activation_level(),
        "data_volume": get_data_volume_metrics(),
        "user_engagement": get_user_engagement_metrics(),
        "system_performance": get_system_performance_metrics(),
        "configuration_completeness": get_configuration_metrics()
    }
    
    return calculate_deployment_health_score(health_metrics)

def get_data_volume_metrics():
    """Measure data volume across critical DocTypes"""
    critical_docs = [
        "Customer", "Supplier", "Item", "Sales Invoice", 
        "Purchase Invoice", "Stock Entry", "Journal Entry"
    ]
    
    metrics = {}
    for doctype in critical_docs:
        metrics[doctype] = {
            "count": frappe.db.count(doctype),
            "created_today": frappe.db.count(doctype, {
                "creation": [">=", frappe.utils.today()]
            }),
            "modified_today": frappe.db.count(doctype, {
                "modified": [">=", frappe.utils.today()]
            })
        }
    
    return metrics

def get_user_engagement_metrics():
    """Measure user engagement and activity"""
    return {
        "active_users_today": frappe.db.count("User", {
            "last_login": [">=", frappe.utils.today()]
        }),
        "active_users_week": frappe.db.count("User", {
            "last_login": [">=", frappe.utils.add_days(frappe.utils.today(), -7)]
        }),
        "total_sessions_today": frappe.db.count("Activity Log", {
            "creation": [">=", frappe.utils.today()],
            "method": "frappe.auth.get_logged_user"
        })
    }
```

### Contextual Help System
*Reference: `/home/frappe/frappe-bench/apps/erpnext/erpnext/utilities/activation.py:66-149`*

```python
def get_help_messages():
    """Returns contextual help based on deployment status"""
    if get_level() > 6:  # Well-established deployment
        return []

    domain = frappe.get_cached_value("Company", erpnext.get_default_company(), "domain")
    messages = []

    message_settings = [
        frappe._dict(
            doctype="Lead",
            title=_("Create Leads"),
            description=_("Leads help you get business, add all your contacts and more as your leads"),
            action=_("Create Lead"),
            route="List/Lead",
            domain=("Manufacturing", "Retail", "Services", "Distribution"),
            target=3,
        ),
        frappe._dict(
            doctype="Sales Order",
            title=_("Manage your orders"),
            description=_("Create Sales Orders to help you plan your work and deliver on-time"),
            action=_("Create Sales Order"),
            route="List/Sales Order",
            domain=("Manufacturing", "Retail", "Services", "Distribution"),
            target=3,
        ),
        # ... additional message configurations
    ]

    # Filter messages based on domain and current progress
    for m in message_settings:
        if not m.domain or domain in m.domain:
            m.count = frappe.db.count(m.doctype)
            if m.count < m.target:
                messages.append(m)

    return messages
```

## Background Job Processing

ERPNext implements comprehensive background processing for resource-intensive operations.

### Bulk Transaction Processing
*Reference: `/home/frappe/frappe-bench/apps/erpnext/erpnext/utilities/bulk_transaction.py:9-27`*

```python
@frappe.whitelist()
def transaction_processing(data, from_doctype, to_doctype):
    """Initiate bulk transaction processing"""
    # Security validation
    frappe.has_permission(from_doctype, "read", throw=True)
    frappe.has_permission(to_doctype, "create", throw=True)

    # Data processing
    if isinstance(data, str):
        deserialized_data = json.loads(data)
    else:
        deserialized_data = data

    length_of_data = len(deserialized_data)

    # User notification
    frappe.msgprint(_("Started a background job to create {1} {0}").format(to_doctype, length_of_data))
    
    # Queue background job
    frappe.enqueue(
        job,
        deserialized_data=deserialized_data,
        from_doctype=from_doctype,
        to_doctype=to_doctype,
    )
```

### Background Job Execution with Error Handling
*Reference: `/home/frappe/frappe-bench/apps/erpnext/erpnext/utilities/bulk_transaction.py:75-96`*

```python
def job(deserialized_data, from_doctype, to_doctype):
    """Execute bulk processing job with comprehensive error handling"""
    fail_count = 0
    
    for d in deserialized_data:
        try:
            doc_name = d.get("name")
            # Create database savepoint for rollback capability
            frappe.db.savepoint("before_creation_state")
            
            # Execute individual task
            task(doc_name, from_doctype, to_doctype)
            
        except Exception:
            # Rollback to savepoint on error
            frappe.db.rollback(save_point="before_creation_state")
            fail_count += 1
            
            # Log failure with detailed error information
            create_log(
                doc_name,
                str(frappe.get_traceback(with_context=True)),
                from_doctype,
                to_doctype,
                status="Failed",
                log_date=str(date.today()),
            )
        else:
            # Log success
            create_log(doc_name, None, from_doctype, to_doctype, 
                      status="Success", log_date=str(date.today()))

    # Provide final status summary
    show_job_status(fail_count, len(deserialized_data), to_doctype)
```

### Transaction Mapping System
*Reference: `/home/frappe/frappe-bench/apps/erpnext/erpnext/utilities/bulk_transaction.py:110-157`*

```python
def task(doc_name, from_doctype, to_doctype):
    """Execute individual transaction mapping"""
    # Import required modules dynamically
    from erpnext.accounts.doctype.payment_entry import payment_entry
    from erpnext.selling.doctype.sales_order import sales_order
    from erpnext.buying.doctype.purchase_order import purchase_order
    # ... additional imports

    # Define transaction mapping
    mapper = {
        "Sales Order": {
            "Sales Invoice": sales_order.make_sales_invoice,
            "Delivery Note": sales_order.make_delivery_note,
            "Payment Entry": payment_entry.get_payment_entry,
        },
        "Purchase Order": {
            "Purchase Invoice": purchase_order.make_purchase_invoice,
            "Purchase Receipt": purchase_order.make_purchase_receipt,
            "Payment Entry": payment_entry.get_payment_entry,
        },
        # ... additional mappings
    }

    # Allow custom extensions via hooks
    hooks = frappe.get_hooks("bulk_transaction_task_mapper")
    for hook in hooks:
        mapper.update(frappe.get_attr(hook)())

    # Execute transaction creation
    frappe.flags.bulk_transaction = True
    
    if to_doctype in ["Payment Entry"]:
        obj = mapper[from_doctype][to_doctype](from_doctype, doc_name)
    else:
        obj = mapper[from_doctype][to_doctype](doc_name)

    # Configure for bulk processing
    obj.flags.ignore_validate = True
    obj.set_title_field()
    obj.insert(ignore_mandatory=True)
    
    del frappe.flags.bulk_transaction
```

### Retry Mechanism for Failed Transactions
*Reference: `/home/frappe/frappe-bench/apps/erpnext/erpnext/utilities/bulk_transaction.py:30-66`*

```python
@frappe.whitelist()
def retry(date: str | None = None):
    """Retry failed transactions from a specific date"""
    if not date:
        date = today()

    failed_docs = frappe.db.get_all(
        "Bulk Transaction Log Detail",
        filters={"date": date, "transaction_status": "Failed", "retried": 0},
        fields=["name", "transaction_name", "from_doctype", "to_doctype"],
    )
    
    if not failed_docs:
        frappe.msgprint(_("There are no Failed transactions"))
    else:
        # Queue retry job
        job = frappe.enqueue(
            retry_failed_transactions,
            failed_docs=failed_docs,
        )
        frappe.msgprint(
            _("Job: {0} has been triggered for processing failed transactions").format(
                get_link_to_form("RQ Job", job.id)
            )
        )

def retry_failed_transactions(failed_docs: list | None):
    """Execute retry logic for failed transactions"""
    if failed_docs:
        for log in failed_docs:
            try:
                # Create savepoint for each retry attempt
                frappe.db.savepoint("before_creation_state")
                task(log.transaction_name, log.from_doctype, log.to_doctype)
            except Exception:
                # Rollback and update log with failure
                frappe.db.rollback(save_point="before_creation_state")
                update_log(log.name, "Failed", 1, str(frappe.get_traceback(with_context=True)))
            else:
                # Update log with success
                update_log(log.name, "Success", 1)
```

## Scheduled Operations Framework

ERPNext implements a comprehensive scheduled operations system for automated maintenance and business processes.

### Scheduler Event Configuration
*Reference: `/home/frappe/frappe-bench/apps/erpnext/erpnext/hooks.py:520-580`*

```python
scheduler_events = {
    "cron": {
        # Every 15 minutes - Critical manufacturing operations
        "0/15 * * * *": [
            "erpnext.manufacturing.doctype.bom_update_log.bom_update_log.resume_bom_cost_update_jobs",
        ],
        # Every 30 minutes - Content updates
        "0/30 * * * *": [
            "erpnext.utilities.doctype.video.video.update_youtube_data",
        ],
        # Hourly offset operations - GL entry optimization
        "30 * * * *": [
            "erpnext.accounts.doctype.gl_entry.gl_entry.rename_gle_sle_docs",
        ],
        # Daily offset operations - Inventory reordering
        "45 0 * * *": [
            "erpnext.stock.reorder_item.reorder_item",
        ],
    },
    "hourly": [
        "erpnext.erpnext_integrations.doctype.plaid_settings.plaid_settings.automatic_synchronization",
        "erpnext.projects.doctype.project.project.project_status_update_reminder",
        "erpnext.projects.doctype.project.project.hourly_reminder",
    ],
    "hourly_long": [
        "erpnext.stock.doctype.repost_item_valuation.repost_item_valuation.repost_entries",
        "erpnext.utilities.bulk_transaction.retry",
    ],
    "daily": [
        # Customer and sales operations
        "erpnext.support.doctype.issue.issue.auto_close_tickets",
        "erpnext.crm.doctype.opportunity.opportunity.auto_close_opportunity",
        "erpnext.controllers.accounts_controller.update_invoice_status",
        
        # Financial operations
        "erpnext.accounts.doctype.fiscal_year.fiscal_year.auto_create_fiscal_year",
        "erpnext.accounts.utils.auto_create_exchange_rate_revaluation_daily",
        "erpnext.accounts.utils.run_ledger_health_checks",
        
        # Asset management
        "erpnext.assets.doctype.asset.asset.update_maintenance_status",
        "erpnext.assets.doctype.asset.asset.make_post_gl_entry",
        
        # Project management
        "erpnext.projects.doctype.task.task.set_tasks_as_overdue",
        "erpnext.projects.doctype.project.project.update_project_sales_billing",
        "erpnext.projects.doctype.project.project.send_project_status_email_to_users",
        
        # Quality and compliance
        "erpnext.quality_management.doctype.quality_review.quality_review.review",
        "erpnext.support.doctype.service_level_agreement.service_level_agreement.check_agreement_status",
        
        # Sales and purchasing
        "erpnext.selling.doctype.quotation.quotation.set_expired_status",
        "erpnext.buying.doctype.supplier_quotation.supplier_quotation.set_expired_status",
        
        # Reporting and analytics
        "erpnext.accounts.doctype.process_statement_of_accounts.process_statement_of_accounts.send_auto_email",
        "erpnext.buying.doctype.supplier_scorecard.supplier_scorecard.refresh_scorecards",
        "erpnext.setup.doctype.company.company.cache_companies_monthly_sales_history",
    ],
    "weekly": [
        "erpnext.accounts.utils.auto_create_exchange_rate_revaluation_weekly",
    ],
    "daily_long": [
        # Resource-intensive daily operations
        "erpnext.accounts.doctype.process_subscription.process_subscription.create_subscription_process",
        "erpnext.setup.doctype.email_digest.email_digest.send",
        "erpnext.manufacturing.doctype.bom_update_tool.bom_update_tool.auto_update_latest_price_in_all_boms",
        "erpnext.crm.utils.open_leads_opportunities_based_on_todays_event",
        "erpnext.assets.doctype.asset.depreciation.post_depreciation_entries",
    ],
    "monthly_long": [
        # Monthly financial processing
        "erpnext.accounts.deferred_revenue.process_deferred_accounting",
        "erpnext.accounts.utils.auto_create_exchange_rate_revaluation_monthly",
    ],
}
```

### Custom Scheduled Task Implementation

```python
def create_custom_scheduled_task():
    """Pattern for creating custom scheduled operations"""
    
    # In hooks.py
    custom_scheduler_events = {
        "daily": [
            "custom_app.operations.daily_cleanup.cleanup_temp_data",
            "custom_app.operations.daily_reports.generate_daily_reports",
            "custom_app.operations.health_checks.run_system_health_check",
        ],
        "hourly": [
            "custom_app.integrations.external_sync.sync_master_data",
            "custom_app.operations.performance.optimize_database_performance",
        ],
        "cron": {
            "0 2 * * *": [  # 2 AM daily
                "custom_app.operations.maintenance.run_database_maintenance",
            ],
            "*/5 * * * *": [  # Every 5 minutes
                "custom_app.monitoring.health_monitor.check_critical_services",
            ]
        }
    }

def run_system_health_check():
    """Comprehensive system health monitoring"""
    try:
        health_report = {
            "database": check_database_health(),
            "queue": check_queue_health(), 
            "storage": check_storage_health(),
            "performance": check_performance_metrics(),
            "integrations": check_integration_health()
        }
        
        # Log health status
        log_health_report(health_report)
        
        # Alert on critical issues
        critical_issues = identify_critical_issues(health_report)
        if critical_issues:
            send_health_alerts(critical_issues)
            
    except Exception as e:
        frappe.log_error("Health check failed", str(e))
        send_critical_alert("Health Check Failed", str(e))

def check_database_health():
    """Monitor database performance and integrity"""
    return {
        "connection_count": frappe.db.sql("SHOW STATUS LIKE 'Threads_connected'")[0][1],
        "slow_queries": frappe.db.sql("SHOW STATUS LIKE 'Slow_queries'")[0][1],
        "table_locks": frappe.db.sql("SHOW STATUS LIKE 'Table_locks_waited'")[0][1],
        "innodb_buffer_pool_usage": get_innodb_buffer_pool_usage(),
        "database_size": get_database_size()
    }
```

## Bulk Operations Management

### Batch Processing Pattern

```python
class BatchProcessor:
    """Generic batch processing framework"""
    
    def __init__(self, operation_name, batch_size=1000):
        self.operation_name = operation_name
        self.batch_size = batch_size
        self.processed_count = 0
        self.failed_count = 0
        self.batch_logs = []
    
    def process_in_batches(self, data_source, processing_function):
        """Process large datasets in manageable batches"""
        total_records = len(data_source) if hasattr(data_source, '__len__') else None
        
        # Initialize progress tracking
        self.start_batch_processing(total_records)
        
        try:
            # Process in batches
            for batch_num, batch_data in enumerate(self.get_batches(data_source)):
                batch_result = self.process_batch(batch_num, batch_data, processing_function)
                self.update_progress(batch_result)
                
                # Optional: Allow breaking on high failure rate
                if self.should_stop_processing():
                    break
                    
        except Exception as e:
            self.handle_processing_error(e)
        finally:
            self.finalize_batch_processing()
    
    def get_batches(self, data_source):
        """Yield data in batches of specified size"""
        if hasattr(data_source, '__iter__'):
            batch = []
            for item in data_source:
                batch.append(item)
                if len(batch) >= self.batch_size:
                    yield batch
                    batch = []
            if batch:  # Yield remaining items
                yield batch
        else:
            # Handle database cursor or other iterable sources
            while True:
                batch = data_source.fetchmany(self.batch_size)
                if not batch:
                    break
                yield batch
    
    def process_batch(self, batch_num, batch_data, processing_function):
        """Process a single batch with error handling"""
        batch_result = {
            "batch_num": batch_num,
            "batch_size": len(batch_data),
            "success_count": 0,
            "error_count": 0,
            "errors": []
        }
        
        # Create batch transaction for rollback capability
        frappe.db.savepoint(f"batch_{batch_num}_start")
        
        try:
            for item in batch_data:
                try:
                    processing_function(item)
                    batch_result["success_count"] += 1
                except Exception as e:
                    batch_result["error_count"] += 1
                    batch_result["errors"].append({
                        "item": str(item),
                        "error": str(e)
                    })
                    
                    # Log individual item error
                    frappe.log_error(f"Batch processing error for {item}", str(e))
            
            # Commit batch if success rate is acceptable
            if self.is_batch_acceptable(batch_result):
                frappe.db.commit()
            else:
                frappe.db.rollback(save_point=f"batch_{batch_num}_start")
                
        except Exception as e:
            frappe.db.rollback(save_point=f"batch_{batch_num}_start")
            batch_result["batch_error"] = str(e)
            
        return batch_result
    
    def is_batch_acceptable(self, batch_result):
        """Determine if batch results are acceptable"""
        error_rate = batch_result["error_count"] / batch_result["batch_size"]
        return error_rate < 0.1  # Accept if less than 10% error rate
```

### Rename Tool Pattern for Bulk Operations
*Reference: `/home/frappe/frappe-bench/apps/erpnext/erpnext/utilities/doctype/rename_tool/rename_tool.py:28-50`*

```python
@frappe.whitelist()
def upload(select_doctype=None, rows=None):
    """Handle bulk rename operations via CSV upload"""
    from frappe.utils.csvutils import read_csv_content_from_attached_file

    if not select_doctype:
        select_doctype = frappe.form_dict.select_doctype

    # Verify permissions
    if not frappe.has_permission(select_doctype, "write"):
        raise frappe.PermissionError

    # Read CSV data
    rows = read_csv_content_from_attached_file(frappe.get_doc("Rename Tool", "Rename Tool"))

    # Process in chunks to prevent memory issues
    # bulk rename allows only 500 rows at a time
    for i in range(0, len(rows), 500):
        chunk_rows = rows[i:i + 500]
        
        # Queue background job for each chunk
        frappe.enqueue(
            process_rename_chunk,
            rows=chunk_rows,
            doctype=select_doctype,
            queue='long'  # Use long queue for potentially time-consuming operations
        )

def process_rename_chunk(rows, doctype):
    """Process a chunk of rename operations"""
    try:
        from frappe.model.rename_doc import bulk_rename
        
        # Validate data before processing
        validated_rows = validate_rename_data(rows, doctype)
        
        # Execute bulk rename
        bulk_rename(doctype, validated_rows)
        
        # Log successful completion
        frappe.log_info(f"Successfully renamed {len(validated_rows)} {doctype} documents")
        
    except Exception as e:
        # Log error and continue with next chunk
        frappe.log_error(f"Rename chunk processing failed for {doctype}", str(e))
        raise

def validate_rename_data(rows, doctype):
    """Validate rename data before processing"""
    validated_rows = []
    
    for row in rows:
        old_name, new_name = row[0], row[1]
        
        # Check if source document exists
        if not frappe.db.exists(doctype, old_name):
            frappe.log_error(f"Source document {old_name} not found", "Rename Validation")
            continue
            
        # Check if target name is already in use
        if frappe.db.exists(doctype, new_name):
            frappe.log_error(f"Target name {new_name} already exists", "Rename Validation")
            continue
            
        validated_rows.append([old_name, new_name])
    
    return validated_rows
```

## System Health Monitoring

### Automated Health Checks

```python
def run_comprehensive_health_check():
    """Execute comprehensive system health assessment"""
    health_components = {
        "database": DatabaseHealthChecker(),
        "queue_system": QueueHealthChecker(),
        "file_system": FileSystemHealthChecker(),
        "memory_usage": MemoryHealthChecker(),
        "integrations": IntegrationHealthChecker(),
        "business_logic": BusinessLogicHealthChecker()
    }
    
    overall_health = {}
    critical_issues = []
    
    for component_name, checker in health_components.items():
        try:
            component_health = checker.check_health()
            overall_health[component_name] = component_health
            
            if component_health.get("status") == "critical":
                critical_issues.extend(component_health.get("issues", []))
                
        except Exception as e:
            overall_health[component_name] = {
                "status": "error",
                "error": str(e)
            }
            critical_issues.append(f"{component_name}: {str(e)}")
    
    # Store health report
    store_health_report(overall_health)
    
    # Handle critical issues
    if critical_issues:
        handle_critical_health_issues(critical_issues)
    
    return overall_health

class DatabaseHealthChecker:
    """Monitor database performance and integrity"""
    
    def check_health(self):
        """Perform comprehensive database health check"""
        checks = {
            "connectivity": self.check_connectivity(),
            "performance": self.check_performance(),
            "integrity": self.check_integrity(),
            "replication": self.check_replication_status(),
            "storage": self.check_storage_usage()
        }
        
        # Determine overall status
        status = "healthy"
        issues = []
        
        for check_name, check_result in checks.items():
            if check_result.get("status") == "critical":
                status = "critical"
                issues.extend(check_result.get("issues", []))
            elif check_result.get("status") == "warning" and status == "healthy":
                status = "warning"
                issues.extend(check_result.get("issues", []))
        
        return {
            "status": status,
            "issues": issues,
            "details": checks,
            "timestamp": frappe.utils.now()
        }
    
    def check_connectivity(self):
        """Test database connectivity"""
        try:
            result = frappe.db.sql("SELECT 1 as test")
            return {
                "status": "healthy",
                "response_time": frappe.utils.now()
            }
        except Exception as e:
            return {
                "status": "critical",
                "issues": [f"Database connectivity failed: {str(e)}"]
            }
    
    def check_performance(self):
        """Monitor database performance metrics"""
        try:
            # Query performance metrics
            slow_queries = frappe.db.sql("SHOW STATUS LIKE 'Slow_queries'")
            connections = frappe.db.sql("SHOW STATUS LIKE 'Threads_connected'")
            
            slow_query_count = int(slow_queries[0][1]) if slow_queries else 0
            connection_count = int(connections[0][1]) if connections else 0
            
            issues = []
            status = "healthy"
            
            if slow_query_count > 100:
                issues.append(f"High slow query count: {slow_query_count}")
                status = "warning"
                
            if connection_count > 100:
                issues.append(f"High connection count: {connection_count}")
                status = "warning"
            
            return {
                "status": status,
                "issues": issues,
                "metrics": {
                    "slow_queries": slow_query_count,
                    "connections": connection_count
                }
            }
            
        except Exception as e:
            return {
                "status": "error",
                "issues": [f"Performance check failed: {str(e)}"]
            }

class QueueHealthChecker:
    """Monitor background job queue health"""
    
    def check_health(self):
        """Check queue system health"""
        try:
            # Check Redis connection
            redis_conn = frappe.cache()
            redis_conn.ping()
            
            # Check queue lengths
            queue_stats = self.get_queue_statistics()
            
            issues = []
            status = "healthy"
            
            # Alert on long queues
            for queue_name, stats in queue_stats.items():
                if stats["pending"] > 1000:
                    issues.append(f"Queue {queue_name} has {stats['pending']} pending jobs")
                    status = "warning"
                
                if stats["failed"] > 100:
                    issues.append(f"Queue {queue_name} has {stats['failed']} failed jobs")
                    status = "warning"
            
            return {
                "status": status,
                "issues": issues,
                "queue_stats": queue_stats
            }
            
        except Exception as e:
            return {
                "status": "critical", 
                "issues": [f"Queue system unavailable: {str(e)}"]
            }
    
    def get_queue_statistics(self):
        """Get detailed queue statistics"""
        import rq
        from rq import Queue
        
        stats = {}
        queue_names = ['default', 'short', 'long']
        
        for queue_name in queue_names:
            try:
                q = Queue(queue_name, connection=frappe.cache())
                stats[queue_name] = {
                    "pending": len(q),
                    "failed": len(q.failed_job_registry),
                    "finished": len(q.finished_job_registry)
                }
            except Exception as e:
                stats[queue_name] = {"error": str(e)}
        
        return stats
```

## Data Migration Patterns

### Migration Framework

```python
class DataMigrator:
    """Framework for safe data migrations"""
    
    def __init__(self, migration_name, source_version, target_version):
        self.migration_name = migration_name
        self.source_version = source_version
        self.target_version = target_version
        self.backup_created = False
        self.migration_log = []
    
    def execute_migration(self):
        """Execute migration with comprehensive safety measures"""
        try:
            # Pre-migration validation
            if not self.validate_pre_conditions():
                raise Exception("Pre-migration validation failed")
            
            # Create backup
            self.create_migration_backup()
            
            # Execute migration steps
            self.run_migration_steps()
            
            # Post-migration validation
            if not self.validate_post_conditions():
                self.rollback_migration()
                raise Exception("Post-migration validation failed")
            
            # Mark migration as complete
            self.mark_migration_complete()
            
        except Exception as e:
            self.handle_migration_error(e)
            raise
    
    def validate_pre_conditions(self):
        """Validate system state before migration"""
        checks = [
            self.check_database_consistency(),
            self.check_system_resources(),
            self.check_user_activity(),
            self.check_dependency_versions()
        ]
        
        return all(checks)
    
    def create_migration_backup(self):
        """Create comprehensive system backup"""
        backup_config = {
            "tables": self.get_affected_tables(),
            "files": self.get_affected_files(),
            "settings": self.get_affected_settings()
        }
        
        backup_id = self.create_backup(backup_config)
        self.backup_created = True
        self.migration_log.append(f"Backup created: {backup_id}")
        
        return backup_id
    
    def run_migration_steps(self):
        """Execute migration steps with transaction management"""
        migration_steps = self.get_migration_steps()
        
        for step_num, step in enumerate(migration_steps):
            try:
                # Create savepoint for step rollback
                savepoint_name = f"migration_step_{step_num}"
                frappe.db.savepoint(savepoint_name)
                
                # Execute step
                self.execute_migration_step(step)
                
                # Log step completion
                self.migration_log.append(f"Step {step_num + 1} completed: {step['name']}")
                
            except Exception as e:
                # Rollback step and re-raise
                frappe.db.rollback(save_point=savepoint_name)
                self.migration_log.append(f"Step {step_num + 1} failed: {str(e)}")
                raise
    
    def get_migration_steps(self):
        """Define migration steps - override in subclasses"""
        return [
            {
                "name": "Schema Changes",
                "function": self.apply_schema_changes,
                "reversible": True
            },
            {
                "name": "Data Transformation", 
                "function": self.transform_data,
                "reversible": False
            },
            {
                "name": "Index Optimization",
                "function": self.optimize_indexes,
                "reversible": True
            }
        ]
```

## Performance Optimization

### Query Optimization Patterns

```python
class QueryOptimizer:
    """Database query optimization utilities"""
    
    @staticmethod
    def optimize_large_table_queries(table_name, batch_size=10000):
        """Optimize queries on large tables using batch processing"""
        
        # Get total record count
        total_count = frappe.db.count(table_name)
        
        if total_count < batch_size:
            # Process normally for small tables
            return frappe.get_all(table_name)
        
        # Process in batches for large tables
        results = []
        offset = 0
        
        while offset < total_count:
            batch = frappe.db.sql(f"""
                SELECT * FROM `tab{table_name}`
                ORDER BY name
                LIMIT {batch_size} OFFSET {offset}
            """, as_dict=True)
            
            results.extend(batch)
            offset += batch_size
            
            # Yield control to prevent timeout
            frappe.db.commit()
        
        return results
    
    @staticmethod
    def create_performance_indexes():
        """Create indexes for improved query performance"""
        
        performance_indexes = [
            {
                "table": "tabSales Invoice",
                "columns": ["posting_date", "customer"],
                "name": "idx_si_posting_customer"
            },
            {
                "table": "tabStock Ledger Entry",
                "columns": ["item_code", "posting_date", "warehouse"],
                "name": "idx_sle_item_date_warehouse"
            },
            {
                "table": "tabGL Entry",
                "columns": ["account", "posting_date", "company"],
                "name": "idx_gle_account_date_company"
            }
        ]
        
        for index in performance_indexes:
            try:
                if not QueryOptimizer.index_exists(index["table"], index["name"]):
                    QueryOptimizer.create_index(index)
                    
            except Exception as e:
                frappe.log_error(f"Index creation failed: {index['name']}", str(e))
    
    @staticmethod
    def index_exists(table_name, index_name):
        """Check if index exists"""
        result = frappe.db.sql(f"""
            SELECT COUNT(*) as count
            FROM INFORMATION_SCHEMA.STATISTICS 
            WHERE TABLE_SCHEMA = DATABASE()
            AND TABLE_NAME = '{table_name}'
            AND INDEX_NAME = '{index_name}'
        """)
        
        return result[0][0] > 0 if result else False
    
    @staticmethod
    def create_index(index_config):
        """Create database index"""
        columns = ", ".join(index_config["columns"])
        
        sql = f"""
            CREATE INDEX {index_config["name"]} 
            ON {index_config["table"]} ({columns})
        """
        
        frappe.db.sql(sql)
        frappe.log_info(f"Created index {index_config['name']} on {index_config['table']}")
```

## Production Deployment Strategies

### Blue-Green Deployment Pattern

```python
class BlueGreenDeployment:
    """Blue-green deployment implementation for ERPNext"""
    
    def __init__(self, deployment_config):
        self.config = deployment_config
        self.current_environment = self.get_current_environment()
        self.target_environment = 'green' if self.current_environment == 'blue' else 'blue'
    
    def execute_deployment(self):
        """Execute blue-green deployment"""
        try:
            # Phase 1: Prepare target environment
            self.prepare_target_environment()
            
            # Phase 2: Deploy to target environment
            self.deploy_to_target()
            
            # Phase 3: Validate target environment
            self.validate_target_environment()
            
            # Phase 4: Switch traffic to target
            self.switch_traffic()
            
            # Phase 5: Validate switch success
            self.validate_switch()
            
            # Phase 6: Cleanup old environment
            self.cleanup_old_environment()
            
        except Exception as e:
            self.handle_deployment_failure(e)
            raise
    
    def prepare_target_environment(self):
        """Prepare the target environment for deployment"""
        steps = [
            self.provision_target_infrastructure,
            self.sync_database_to_target,
            self.sync_files_to_target,
            self.configure_target_environment
        ]
        
        for step in steps:
            step()
    
    def deploy_to_target(self):
        """Deploy application to target environment"""
        # Pull latest code
        self.pull_application_code()
        
        # Install dependencies
        self.install_dependencies()
        
        # Run database migrations
        self.run_database_migrations()
        
        # Update configurations
        self.update_configurations()
        
        # Start services
        self.start_target_services()
    
    def validate_target_environment(self):
        """Comprehensive validation of target environment"""
        validation_checks = [
            self.check_application_health,
            self.check_database_connectivity,
            self.check_queue_system,
            self.check_file_access,
            self.run_smoke_tests
        ]
        
        for check in validation_checks:
            if not check():
                raise Exception(f"Validation failed: {check.__name__}")
    
    def switch_traffic(self):
        """Switch user traffic to target environment"""
        # Update load balancer configuration
        self.update_load_balancer()
        
        # Update DNS if needed
        self.update_dns_records()
        
        # Wait for propagation
        self.wait_for_traffic_switch()
    
    def rollback_deployment(self):
        """Rollback deployment in case of failure"""
        # Switch traffic back to previous environment
        self.switch_traffic_back()
        
        # Cleanup failed deployment
        self.cleanup_failed_deployment()
        
        # Restore previous state
        self.restore_previous_state()
```

### Zero-Downtime Migration

```python
def execute_zero_downtime_migration():
    """Execute database migration with zero downtime"""
    
    # Phase 1: Shadow table creation
    create_shadow_tables()
    
    # Phase 2: Dual-write setup
    setup_dual_write_triggers()
    
    # Phase 3: Background migration
    migrate_existing_data()
    
    # Phase 4: Validation
    validate_migrated_data()
    
    # Phase 5: Atomic switch
    perform_atomic_switch()
    
    # Phase 6: Cleanup
    cleanup_migration_artifacts()

def create_shadow_tables():
    """Create shadow tables with new schema"""
    migration_tables = get_migration_table_list()
    
    for table in migration_tables:
        shadow_table_name = f"{table}_shadow"
        
        # Create shadow table with new schema
        create_table_sql = generate_shadow_table_schema(table)
        frappe.db.sql(create_table_sql)
        
        frappe.log_info(f"Created shadow table: {shadow_table_name}")

def setup_dual_write_triggers():
    """Setup database triggers for dual writing"""
    migration_tables = get_migration_table_list()
    
    for table in migration_tables:
        # Create INSERT trigger
        create_insert_trigger(table)
        
        # Create UPDATE trigger  
        create_update_trigger(table)
        
        # Create DELETE trigger
        create_delete_trigger(table)
        
        frappe.log_info(f"Setup dual-write triggers for: {table}")

def migrate_existing_data():
    """Migrate existing data to shadow tables"""
    migration_tables = get_migration_table_list()
    
    for table in migration_tables:
        # Get total record count
        total_records = frappe.db.count(table)
        
        # Migrate in batches
        batch_size = 10000
        for offset in range(0, total_records, batch_size):
            migrate_batch(table, offset, batch_size)
            
            # Show progress
            progress = min(offset + batch_size, total_records)
            frappe.publish_realtime(
                'migration_progress',
                {'table': table, 'progress': progress, 'total': total_records},
                user=frappe.session.user
            )
```

## Best Practices

### Operational Excellence Checklist

```python
def operational_excellence_audit():
    """Comprehensive operational excellence assessment"""
    
    audit_areas = {
        "deployment": {
            "automated_deployment": check_automated_deployment(),
            "blue_green_capability": check_blue_green_setup(),
            "rollback_procedures": check_rollback_procedures(),
            "environment_parity": check_environment_parity()
        },
        "monitoring": {
            "health_monitoring": check_health_monitoring(),
            "performance_monitoring": check_performance_monitoring(),
            "business_monitoring": check_business_monitoring(),
            "alerting_system": check_alerting_system()
        },
        "backup_recovery": {
            "automated_backups": check_automated_backups(),
            "backup_testing": check_backup_testing(),
            "recovery_procedures": check_recovery_procedures(),
            "disaster_recovery": check_disaster_recovery_plan()
        },
        "security": {
            "access_controls": check_access_controls(),
            "security_monitoring": check_security_monitoring(),
            "vulnerability_management": check_vulnerability_management(),
            "compliance_monitoring": check_compliance_monitoring()
        },
        "performance": {
            "query_optimization": check_query_optimization(),
            "caching_strategy": check_caching_strategy(),
            "resource_utilization": check_resource_utilization(),
            "scaling_capability": check_scaling_capability()
        }
    }
    
    return generate_excellence_report(audit_areas)

def check_automated_deployment():
    """Verify automated deployment capabilities"""
    checks = [
        check_ci_cd_pipeline(),
        check_deployment_scripts(),
        check_environment_configs(),
        check_deployment_validation()
    ]
    
    return {
        "status": "pass" if all(checks) else "fail",
        "details": checks,
        "recommendations": generate_deployment_recommendations(checks)
    }

def generate_operational_runbook():
    """Generate comprehensive operational runbook"""
    
    runbook_sections = {
        "deployment_procedures": {
            "standard_deployment": get_deployment_procedure(),
            "hotfix_deployment": get_hotfix_procedure(),
            "rollback_procedure": get_rollback_procedure(),
            "emergency_procedures": get_emergency_procedures()
        },
        "monitoring_procedures": {
            "health_check_procedures": get_health_check_procedures(),
            "performance_monitoring": get_performance_monitoring_procedures(),
            "alert_response": get_alert_response_procedures(),
            "incident_management": get_incident_management_procedures()
        },
        "maintenance_procedures": {
            "regular_maintenance": get_regular_maintenance_procedures(),
            "database_maintenance": get_database_maintenance_procedures(),
            "system_updates": get_system_update_procedures(),
            "capacity_planning": get_capacity_planning_procedures()
        }
    }
    
    return create_runbook_document(runbook_sections)
```

This comprehensive analysis of ERPNext's deployment and operations patterns provides proven approaches for building reliable, scalable, and maintainable production systems that can handle enterprise-scale operations while maintaining high availability and performance.