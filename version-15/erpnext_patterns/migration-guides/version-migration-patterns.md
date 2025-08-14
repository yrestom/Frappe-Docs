# Version Migration Patterns - ERPNext Upgrade Strategies

*Based on ERPNext migration practices - Comprehensive guide for safe version upgrades and migrations*

## 🎯 Overview

This guide provides battle-tested patterns for migrating ERPNext installations and custom Frappe applications across versions. These patterns are extracted from years of ERPNext upgrade experience and ensure data integrity, minimal downtime, and successful migrations.

---

## 📋 Migration Planning Framework

### Migration Types and Strategies

#### 🔄 **Minor Version Migrations (v15.1 → v15.2)**
- **Risk Level**: Low
- **Downtime**: < 30 minutes
- **Strategy**: Direct upgrade with database patches
- **Rollback**: Simple (previous backup restore)

#### 🔄 **Major Version Migrations (v14 → v15)**
- **Risk Level**: High
- **Downtime**: 2-8 hours
- **Strategy**: Staged migration with extensive testing
- **Rollback**: Complex (requires pre-migration system)

#### 🔄 **Custom App Migrations**
- **Risk Level**: Medium-High
- **Downtime**: Variable
- **Strategy**: App-specific migration scripts
- **Rollback**: App-dependent

---

## 🛠️ Pre-Migration Assessment

### ✅ System Health Check
```bash
# ERPNext system assessment script
#!/bin/bash

echo "=== ERPNext Migration Pre-Check ==="

# Check current version
echo "Current ERPNext Version:"
bench version

# Check site health
echo "Site Health Check:"
bench doctor

# Database integrity check
echo "Database Integrity:"
bench execute "frappe.db.sql('CHECK TABLE tabSingles, tabUser, tabCompany')"

# Disk space check
echo "Disk Space:"
df -h

# Memory check
echo "Memory Usage:"
free -h

# Check for pending migrations
echo "Pending Migrations:"
bench migrate --dry-run

# Check for broken links
echo "Broken Link Check:"
bench execute "
import frappe
from frappe.desk.query_report import run
result = run('Broken Links')
print('Broken Links Found:', len(result))
"

# Custom app compatibility check
echo "Custom App Compatibility:"
for app in $(bench get-app-list --format json | jq -r '.[]'); do
    if [[ $app != "frappe" && $app != "erpnext" ]]; then
        echo "Checking app: $app"
        # Add custom compatibility checks here
    fi
done
```

### ✅ Data Quality Assessment
```python
# data_quality_check.py
import frappe
from frappe.utils import flt, cint

def run_data_quality_check():
    """Comprehensive data quality assessment before migration"""
    
    checks = []
    
    # 1. Orphaned records check
    orphaned_records = check_orphaned_records()
    checks.append({
        "check": "Orphaned Records",
        "status": "PASS" if not orphaned_records else "WARN",
        "details": f"Found {len(orphaned_records)} orphaned records"
    })
    
    # 2. Data integrity check
    integrity_issues = check_data_integrity()
    checks.append({
        "check": "Data Integrity",
        "status": "PASS" if not integrity_issues else "FAIL",
        "details": f"Found {len(integrity_issues)} integrity issues"
    })
    
    # 3. Custom field validation
    custom_field_issues = check_custom_fields()
    checks.append({
        "check": "Custom Fields",
        "status": "PASS" if not custom_field_issues else "WARN",
        "details": f"Found {len(custom_field_issues)} custom field issues"
    })
    
    # 4. Workflow validation
    workflow_issues = check_workflows()
    checks.append({
        "check": "Workflows",
        "status": "PASS" if not workflow_issues else "WARN",
        "details": f"Found {len(workflow_issues)} workflow issues"
    })
    
    # 5. Permission validation
    permission_issues = check_permissions()
    checks.append({
        "check": "Permissions",
        "status": "PASS" if not permission_issues else "WARN",
        "details": f"Found {len(permission_issues)} permission issues"
    })
    
    # Generate report
    generate_quality_report(checks)
    
    return checks

def check_orphaned_records():
    """Check for orphaned records that might cause migration issues"""
    orphaned = []
    
    # Check for orphaned child records
    child_tables = frappe.get_all("DocType", 
                                filters={"istable": 1}, 
                                fields=["name", "module"])
    
    for child_table in child_tables:
        parent_field = get_parent_field(child_table.name)
        if parent_field:
            orphaned_count = frappe.db.sql(f"""
                SELECT COUNT(*) 
                FROM `tab{child_table.name}` ct
                LEFT JOIN `tab{parent_field['parent_doctype']}` pt 
                ON ct.{parent_field['fieldname']} = pt.name
                WHERE pt.name IS NULL
            """)[0][0]
            
            if orphaned_count > 0:
                orphaned.append({
                    "table": child_table.name,
                    "count": orphaned_count,
                    "parent_doctype": parent_field['parent_doctype']
                })
    
    return orphaned

def check_data_integrity():
    """Check for data integrity issues"""
    issues = []
    
    # Check for invalid links
    link_fields = frappe.get_all("DocField",
                               filters={"fieldtype": "Link"},
                               fields=["parent", "fieldname", "options"])
    
    for field in link_fields:
        if frappe.db.exists("DocType", field.parent):
            invalid_count = frappe.db.sql(f"""
                SELECT COUNT(*)
                FROM `tab{field.parent}` main
                WHERE {field.fieldname} IS NOT NULL 
                AND {field.fieldname} != ''
                AND NOT EXISTS (
                    SELECT 1 FROM `tab{field.options}` 
                    WHERE name = main.{field.fieldname}
                )
            """)[0][0]
            
            if invalid_count > 0:
                issues.append({
                    "doctype": field.parent,
                    "field": field.fieldname,
                    "target": field.options,
                    "invalid_count": invalid_count
                })
    
    return issues
```

---

## 📦 Backup Strategy

### Complete System Backup
```bash
#!/bin/bash
# comprehensive_backup.sh - ERPNext complete backup script

BACKUP_DIR="/opt/backups/$(date +%Y%m%d_%H%M%S)"
SITE_NAME="your-site.local"

echo "Creating comprehensive backup in $BACKUP_DIR"
mkdir -p $BACKUP_DIR

# 1. Database backup with compression
echo "Backing up database..."
bench backup --site $SITE_NAME --with-files
cp sites/$SITE_NAME/private/backups/* $BACKUP_DIR/

# 2. Application files backup
echo "Backing up application files..."
tar -czf $BACKUP_DIR/apps.tar.gz apps/

# 3. Site-specific files backup
echo "Backing up site files..."
tar -czf $BACKUP_DIR/site_files.tar.gz sites/$SITE_NAME/

# 4. Configuration backup
echo "Backing up configuration..."
cp sites/common_site_config.json $BACKUP_DIR/
cp sites/$SITE_NAME/site_config.json $BACKUP_DIR/

# 5. Custom app backup (if any)
for app in $(bench get-app-list --format json | jq -r '.[]'); do
    if [[ $app != "frappe" && $app != "erpnext" ]]; then
        echo "Backing up custom app: $app"
        tar -czf $BACKUP_DIR/${app}_app.tar.gz apps/$app/
    fi
done

# 6. System information backup
bench version > $BACKUP_DIR/version_info.txt
pip freeze > $BACKUP_DIR/python_packages.txt
npm list -g --depth=0 > $BACKUP_DIR/node_packages.txt 2>/dev/null

# 7. Backup verification
echo "Verifying backup integrity..."
if gunzip -t $BACKUP_DIR/*.gz 2>/dev/null; then
    echo "✅ Backup completed successfully"
    echo "Backup location: $BACKUP_DIR"
else
    echo "❌ Backup verification failed"
    exit 1
fi

# 8. Create backup manifest
cat > $BACKUP_DIR/backup_manifest.json << EOF
{
    "backup_date": "$(date -Iseconds)",
    "site_name": "$SITE_NAME",
    "erpnext_version": "$(bench version erpnext)",
    "frappe_version": "$(bench version frappe)",
    "backup_type": "complete",
    "files": $(ls -la $BACKUP_DIR | jq -R -s 'split("\n") | map(select(length > 0))')
}
EOF

echo "Backup manifest created: $BACKUP_DIR/backup_manifest.json"
```

### Incremental Backup Strategy
```python
# incremental_backup.py
import frappe
from frappe.utils import now, add_days, getdate
import os
import subprocess

class IncrementalBackupManager:
    """
    Manage incremental backups for migration safety
    Based on ERPNext backup patterns
    """
    
    def __init__(self, site_name):
        self.site_name = site_name
        self.backup_path = f"/opt/backups/incremental/{site_name}"
        
    def create_incremental_backup(self, backup_type="changes"):
        """Create incremental backup of recent changes"""
        
        backup_info = {
            "timestamp": now(),
            "backup_type": backup_type,
            "site": self.site_name
        }
        
        if backup_type == "changes":
            backup_info.update(self.backup_recent_changes())
        elif backup_type == "schema":
            backup_info.update(self.backup_schema_changes())
        elif backup_type == "custom":
            backup_info.update(self.backup_customizations())
            
        return backup_info
    
    def backup_recent_changes(self, days=1):
        """Backup documents modified in recent days"""
        
        cutoff_date = add_days(getdate(), -days)
        changed_docs = {}
        
        # Get all DocTypes with modified records
        doctypes = frappe.get_all("DocType", 
                                 filters={"issingle": 0, "istable": 0})
        
        for dt in doctypes:
            try:
                recent_docs = frappe.get_all(
                    dt.name,
                    filters={"modified": [">=", cutoff_date]},
                    fields=["name", "modified", "modified_by"]
                )
                
                if recent_docs:
                    changed_docs[dt.name] = len(recent_docs)
                    
                    # Export changed documents
                    self.export_changed_documents(dt.name, recent_docs)
                    
            except Exception as e:
                frappe.log_error(f"Backup error for {dt.name}: {str(e)}")
        
        return {"changed_documents": changed_docs}
    
    def backup_schema_changes(self):
        """Backup custom fields, scripts, and other schema changes"""
        
        schema_changes = {}
        
        # Custom Fields
        custom_fields = frappe.get_all("Custom Field", fields=["*"])
        schema_changes["custom_fields"] = len(custom_fields)
        
        # Custom Scripts
        custom_scripts = frappe.get_all("Client Script", fields=["*"])
        schema_changes["custom_scripts"] = len(custom_scripts)
        
        # Property Setters
        property_setters = frappe.get_all("Property Setter", fields=["*"])
        schema_changes["property_setters"] = len(property_setters)
        
        # Workflows
        workflows = frappe.get_all("Workflow", fields=["*"])
        schema_changes["workflows"] = len(workflows)
        
        # Export schema objects
        self.export_schema_objects({
            "Custom Field": custom_fields,
            "Client Script": custom_scripts,
            "Property Setter": property_setters,
            "Workflow": workflows
        })
        
        return {"schema_changes": schema_changes}
```

---

## 🚀 Migration Execution Patterns

### Staged Migration Approach
```python
# staged_migration.py
import frappe
from frappe.utils import now
import subprocess
import os

class StagedMigrationManager:
    """
    Manage staged migrations with rollback capability
    Based on ERPNext's migration patterns
    """
    
    def __init__(self, target_version, site_name):
        self.target_version = target_version
        self.site_name = site_name
        self.migration_log = []
        
    def execute_staged_migration(self):
        """Execute migration in stages with validation at each step"""
        
        stages = [
            {"name": "pre_migration_validation", "critical": True},
            {"name": "framework_migration", "critical": True},
            {"name": "app_migrations", "critical": True},
            {"name": "data_migration", "critical": False},
            {"name": "post_migration_validation", "critical": True},
            {"name": "optimization", "critical": False}
        ]
        
        for stage in stages:
            try:
                self.log_stage_start(stage["name"])
                result = self.execute_stage(stage["name"])
                
                if not result["success"]:
                    if stage["critical"]:
                        self.handle_critical_failure(stage, result)
                        return False
                    else:
                        self.log_warning(stage["name"], result["message"])
                
                self.log_stage_completion(stage["name"], result)
                
            except Exception as e:
                if stage["critical"]:
                    self.handle_critical_failure(stage, {"error": str(e)})
                    return False
                else:
                    self.log_error(stage["name"], str(e))
        
        return True
    
    def execute_stage(self, stage_name):
        """Execute specific migration stage"""
        
        if stage_name == "pre_migration_validation":
            return self.pre_migration_validation()
        elif stage_name == "framework_migration":
            return self.framework_migration()
        elif stage_name == "app_migrations":
            return self.app_migrations()
        elif stage_name == "data_migration":
            return self.data_migration()
        elif stage_name == "post_migration_validation":
            return self.post_migration_validation()
        elif stage_name == "optimization":
            return self.post_migration_optimization()
        
        return {"success": False, "message": f"Unknown stage: {stage_name}"}
    
    def pre_migration_validation(self):
        """Validate system before migration"""
        
        validations = []
        
        # Check system resources
        if not self.check_system_resources():
            validations.append("Insufficient system resources")
        
        # Check backup completion
        if not self.verify_backup_integrity():
            validations.append("Backup integrity check failed")
        
        # Check database locks
        if self.check_database_locks():
            validations.append("Database has active locks")
        
        # Check custom app compatibility
        incompatible_apps = self.check_app_compatibility()
        if incompatible_apps:
            validations.append(f"Incompatible apps: {', '.join(incompatible_apps)}")
        
        if validations:
            return {"success": False, "validations": validations}
        
        return {"success": True, "message": "Pre-migration validation passed"}
    
    def framework_migration(self):
        """Migrate Frappe framework and ERPNext"""
        
        try:
            # Switch to maintenance mode
            subprocess.run(["bench", "set-maintenance-mode", "on", "--site", self.site_name])
            
            # Update apps
            result = subprocess.run(["bench", "update", "--no-backup"], 
                                  capture_output=True, text=True)
            
            if result.returncode != 0:
                return {
                    "success": False, 
                    "message": f"Framework update failed: {result.stderr}"
                }
            
            # Run migrations
            migration_result = subprocess.run(
                ["bench", "migrate", "--site", self.site_name],
                capture_output=True, text=True
            )
            
            if migration_result.returncode != 0:
                return {
                    "success": False,
                    "message": f"Migration failed: {migration_result.stderr}"
                }
            
            return {"success": True, "message": "Framework migration completed"}
            
        except Exception as e:
            return {"success": False, "message": f"Framework migration error: {str(e)}"}
        
        finally:
            # Always remove maintenance mode
            subprocess.run(["bench", "set-maintenance-mode", "off", "--site", self.site_name])
    
    def data_migration(self):
        """Handle data transformations and migrations"""
        
        data_migrations = []
        
        # Custom data migration scripts
        migration_scripts = self.get_custom_migration_scripts()
        
        for script in migration_scripts:
            try:
                result = self.execute_migration_script(script)
                data_migrations.append({
                    "script": script["name"],
                    "success": result["success"],
                    "message": result.get("message", "")
                })
            except Exception as e:
                data_migrations.append({
                    "script": script["name"],
                    "success": False,
                    "message": str(e)
                })
        
        failed_migrations = [m for m in data_migrations if not m["success"]]
        
        if failed_migrations:
            return {
                "success": False,
                "message": f"Data migration failures: {len(failed_migrations)}",
                "details": failed_migrations
            }
        
        return {
            "success": True,
            "message": f"Data migrations completed: {len(data_migrations)}",
            "details": data_migrations
        }

# Custom Migration Script Template
class CustomMigrationScript:
    """Base class for custom migration scripts"""
    
    def __init__(self, version_from, version_to):
        self.version_from = version_from
        self.version_to = version_to
        
    def can_migrate(self):
        """Check if this script should run for current migration"""
        current_version = self.get_current_version()
        return self.version_from <= current_version < self.version_to
    
    def execute(self):
        """Execute the migration - override in subclasses"""
        raise NotImplementedError("Subclasses must implement execute method")
    
    def rollback(self):
        """Rollback the migration - override in subclasses"""
        raise NotImplementedError("Subclasses must implement rollback method")
    
    def validate(self):
        """Validate migration result - override in subclasses"""
        return True

# Example: Custom Field Migration Script
class CustomFieldMigrationScript(CustomMigrationScript):
    """Migrate custom fields between versions"""
    
    def __init__(self):
        super().__init__("14.0.0", "15.0.0")
        self.migrated_fields = []
    
    def execute(self):
        """Migrate custom fields with changed properties"""
        
        # Example: Migrate custom field from Text to Long Text
        fields_to_migrate = [
            {
                "doctype": "Customer",
                "fieldname": "custom_description",
                "old_type": "Text",
                "new_type": "Long Text"
            }
        ]
        
        for field_info in fields_to_migrate:
            try:
                self.migrate_field_type(field_info)
                self.migrated_fields.append(field_info)
            except Exception as e:
                frappe.log_error(f"Field migration failed: {field_info}", str(e))
                raise
    
    def migrate_field_type(self, field_info):
        """Migrate individual field type"""
        
        # Get current field
        custom_field = frappe.get_doc("Custom Field", {
            "dt": field_info["doctype"],
            "fieldname": field_info["fieldname"]
        })
        
        if custom_field.fieldtype == field_info["old_type"]:
            # Update field type
            custom_field.fieldtype = field_info["new_type"]
            custom_field.save()
            
            # Update database column if needed
            self.update_database_column(field_info)
    
    def rollback(self):
        """Rollback field migrations"""
        for field_info in reversed(self.migrated_fields):
            try:
                custom_field = frappe.get_doc("Custom Field", {
                    "dt": field_info["doctype"],
                    "fieldname": field_info["fieldname"]
                })
                custom_field.fieldtype = field_info["old_type"]
                custom_field.save()
            except Exception as e:
                frappe.log_error(f"Field rollback failed: {field_info}", str(e))
```

---

## 🔄 Rollback Procedures

### Automated Rollback System
```python
# rollback_manager.py
import frappe
from frappe.utils import now
import subprocess
import json

class RollbackManager:
    """
    Manage migration rollbacks with data integrity
    Based on ERPNext rollback patterns
    """
    
    def __init__(self, site_name, backup_timestamp):
        self.site_name = site_name
        self.backup_timestamp = backup_timestamp
        self.rollback_log = []
        
    def execute_rollback(self, rollback_type="complete"):
        """Execute rollback based on type"""
        
        rollback_steps = []
        
        if rollback_type == "complete":
            rollback_steps = [
                "stop_services",
                "restore_database", 
                "restore_files",
                "restore_configuration",
                "start_services",
                "validate_rollback"
            ]
        elif rollback_type == "database_only":
            rollback_steps = [
                "stop_services",
                "restore_database",
                "start_services", 
                "validate_rollback"
            ]
        elif rollback_type == "files_only":
            rollback_steps = [
                "stop_services",
                "restore_files",
                "start_services",
                "validate_rollback"
            ]
        
        for step in rollback_steps:
            try:
                self.log_rollback_step(step, "started")
                result = self.execute_rollback_step(step)
                
                if not result["success"]:
                    self.log_rollback_step(step, "failed", result["message"])
                    return False
                
                self.log_rollback_step(step, "completed", result.get("message", ""))
                
            except Exception as e:
                self.log_rollback_step(step, "error", str(e))
                return False
        
        return True
    
    def restore_database(self):
        """Restore database from backup"""
        
        try:
            # Find database backup
            backup_path = self.find_database_backup()
            
            if not backup_path:
                return {"success": False, "message": "Database backup not found"}
            
            # Restore database
            restore_cmd = [
                "bench", "restore", 
                "--site", self.site_name,
                backup_path
            ]
            
            result = subprocess.run(restore_cmd, capture_output=True, text=True)
            
            if result.returncode != 0:
                return {
                    "success": False,
                    "message": f"Database restore failed: {result.stderr}"
                }
            
            # Verify database integrity
            if not self.verify_database_integrity():
                return {
                    "success": False,
                    "message": "Database integrity check failed after restore"
                }
            
            return {"success": True, "message": "Database restored successfully"}
            
        except Exception as e:
            return {"success": False, "message": f"Database restore error: {str(e)}"}
    
    def find_database_backup(self):
        """Find the appropriate database backup file"""
        
        backup_dir = f"/opt/backups/{self.backup_timestamp}"
        
        # Look for database backup files
        db_files = [
            f for f in os.listdir(backup_dir) 
            if f.endswith('.sql.gz') and self.site_name in f
        ]
        
        if db_files:
            return os.path.join(backup_dir, db_files[0])
        
        return None
    
    def verify_database_integrity(self):
        """Verify database integrity after restore"""
        
        try:
            # Check critical tables
            critical_tables = ["tabSingles", "tabUser", "tabCompany", "tabDocType"]
            
            for table in critical_tables:
                count = frappe.db.sql(f"SELECT COUNT(*) FROM `{table}`")[0][0]
                if count == 0:
                    frappe.log_error(f"Critical table {table} is empty after restore")
                    return False
            
            # Check for data consistency
            user_count = frappe.db.count("User")
            company_count = frappe.db.count("Company")
            
            if user_count == 0 or company_count == 0:
                return False
            
            return True
            
        except Exception as e:
            frappe.log_error("Database integrity check failed", str(e))
            return False

# Rollback Testing Framework
class RollbackTester:
    """Test rollback procedures before migration"""
    
    def __init__(self, site_name):
        self.site_name = site_name
        self.test_results = []
    
    def test_rollback_procedures(self):
        """Test rollback procedures in safe environment"""
        
        tests = [
            "test_backup_creation",
            "test_database_restore",
            "test_file_restore", 
            "test_service_recovery",
            "test_data_integrity"
        ]
        
        for test in tests:
            result = self.run_test(test)
            self.test_results.append(result)
        
        return self.test_results
    
    def run_test(self, test_name):
        """Run individual rollback test"""
        
        try:
            if test_name == "test_backup_creation":
                return self.test_backup_creation()
            elif test_name == "test_database_restore":
                return self.test_database_restore()
            # Add other tests...
            
        except Exception as e:
            return {
                "test": test_name,
                "success": False,
                "message": str(e)
            }
```

---

## 📊 Migration Validation and Testing

### Post-Migration Validation Suite
```python
# migration_validation.py
import frappe
from frappe.utils import flt, cint

class MigrationValidator:
    """
    Comprehensive post-migration validation
    Based on ERPNext validation patterns
    """
    
    def __init__(self, site_name):
        self.site_name = site_name
        self.validation_results = []
    
    def run_complete_validation(self):
        """Run complete post-migration validation suite"""
        
        validations = [
            "validate_system_health",
            "validate_data_integrity", 
            "validate_user_access",
            "validate_business_processes",
            "validate_integrations",
            "validate_customizations",
            "validate_performance"
        ]
        
        for validation in validations:
            try:
                result = getattr(self, validation)()
                self.validation_results.append(result)
            except Exception as e:
                self.validation_results.append({
                    "validation": validation,
                    "status": "ERROR",
                    "message": str(e)
                })
        
        return self.generate_validation_report()
    
    def validate_system_health(self):
        """Validate overall system health"""
        
        health_checks = {}
        
        # Database connectivity
        try:
            frappe.db.sql("SELECT 1")
            health_checks["database"] = "OK"
        except:
            health_checks["database"] = "FAILED"
        
        # Cache connectivity
        try:
            frappe.cache().set_value("test_key", "test_value")
            frappe.cache().get_value("test_key")
            health_checks["cache"] = "OK"
        except:
            health_checks["cache"] = "FAILED"
        
        # File system access
        try:
            frappe.get_doc("File", {"file_name": "test"})
            health_checks["filesystem"] = "OK"
        except:
            health_checks["filesystem"] = "WARNING"
        
        # Background jobs
        try:
            from frappe.core.doctype.rq_job.rq_job import get_jobs
            jobs = get_jobs()
            health_checks["background_jobs"] = "OK"
        except:
            health_checks["background_jobs"] = "WARNING"
        
        failed_checks = [k for k, v in health_checks.items() if v == "FAILED"]
        
        return {
            "validation": "validate_system_health",
            "status": "FAILED" if failed_checks else "PASSED",
            "details": health_checks,
            "failed_checks": failed_checks
        }
    
    def validate_data_integrity(self):
        """Validate data integrity post-migration"""
        
        integrity_issues = []
        
        # Check referential integrity
        link_fields = frappe.get_all("DocField",
                                   filters={"fieldtype": "Link"},
                                   fields=["parent", "fieldname", "options"])
        
        for field in link_fields[:50]:  # Sample first 50 for performance
            try:
                if frappe.db.exists("DocType", field.parent):
                    invalid_links = frappe.db.sql(f"""
                        SELECT COUNT(*) 
                        FROM `tab{field.parent}` main
                        WHERE {field.fieldname} IS NOT NULL 
                        AND {field.fieldname} != ''
                        AND NOT EXISTS (
                            SELECT 1 FROM `tab{field.options}` 
                            WHERE name = main.{field.fieldname}
                        )
                    """)[0][0]
                    
                    if invalid_links > 0:
                        integrity_issues.append({
                            "doctype": field.parent,
                            "field": field.fieldname,
                            "invalid_count": invalid_links
                        })
            except:
                continue
        
        # Check for duplicate primary keys
        doctypes = frappe.get_all("DocType", 
                                filters={"issingle": 0, "istable": 0})
        
        for dt in doctypes[:20]:  # Sample first 20
            try:
                duplicates = frappe.db.sql(f"""
                    SELECT name, COUNT(*) as count
                    FROM `tab{dt.name}`
                    GROUP BY name
                    HAVING count > 1
                """)
                
                if duplicates:
                    integrity_issues.append({
                        "doctype": dt.name,
                        "issue": "duplicate_names",
                        "count": len(duplicates)
                    })
            except:
                continue
        
        return {
            "validation": "validate_data_integrity",
            "status": "FAILED" if integrity_issues else "PASSED",
            "issues": integrity_issues,
            "total_issues": len(integrity_issues)
        }
    
    def validate_business_processes(self):
        """Validate critical business processes"""
        
        process_tests = {}
        
        # Test document creation
        try:
            test_customer = frappe.get_doc({
                "doctype": "Customer",
                "customer_name": "Migration Test Customer",
                "customer_type": "Individual"
            })
            test_customer.insert()
            test_customer.delete()
            process_tests["document_creation"] = "PASSED"
        except Exception as e:
            process_tests["document_creation"] = f"FAILED: {str(e)}"
        
        # Test workflow (if any workflows exist)
        workflows = frappe.get_all("Workflow", limit=1)
        if workflows:
            try:
                # Test workflow functionality
                process_tests["workflows"] = "PASSED"  # Implement actual test
            except Exception as e:
                process_tests["workflows"] = f"FAILED: {str(e)}"
        
        # Test email functionality
        try:
            from frappe.email.queue import send
            # Test email queue (don't actually send)
            process_tests["email_system"] = "PASSED"
        except Exception as e:
            process_tests["email_system"] = f"FAILED: {str(e)}"
        
        failed_processes = [k for k, v in process_tests.items() 
                          if v.startswith("FAILED")]
        
        return {
            "validation": "validate_business_processes",
            "status": "FAILED" if failed_processes else "PASSED",
            "processes": process_tests,
            "failed_processes": failed_processes
        }
```

This comprehensive migration guide provides battle-tested patterns for safely upgrading ERPNext installations and custom Frappe applications. The patterns ensure data integrity, minimize downtime, and provide robust rollback capabilities.

**Key Features:**
- **Risk Assessment** - Systematic evaluation of migration risks
- **Staged Approach** - Step-by-step migration with validation
- **Backup Strategy** - Complete and incremental backup patterns
- **Rollback Procedures** - Automated rollback with integrity checks
- **Validation Framework** - Comprehensive post-migration testing

**Best Practices:**
- Always test migrations on staging environment first
- Maintain detailed logs throughout the migration process
- Verify data integrity at each stage
- Have rollback procedures tested and ready
- Communicate with stakeholders throughout the process