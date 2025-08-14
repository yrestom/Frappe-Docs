# Troubleshooting Guide - Complete Reference

> **Comprehensive troubleshooting guide for common issues, debugging techniques, and error resolution procedures**

## Table of Contents

- [Diagnostic Tools and Techniques](#diagnostic-tools-and-techniques)
- [Application Issues](#application-issues)
- [Database Problems](#database-problems)
- [Performance Issues](#performance-issues)
- [Authentication and Permissions](#authentication-and-permissions)
- [Background Jobs and Scheduler](#background-jobs-and-scheduler)
- [Email and Notifications](#email-and-notifications)
- [File Upload and Storage](#file-upload-and-storage)
- [Installation and Deployment](#installation-and-deployment)
- [Network and Connectivity](#network-and-connectivity)
- [Security Issues](#security-issues)
- [Development and Customization](#development-and-customization)
- [System Administration](#system-administration)
- [Log Analysis](#log-analysis)
- [Recovery Procedures](#recovery-procedures)

## Diagnostic Tools and Techniques

### Built-in Debugging Tools

```python
# debugging/diagnostic_tools.py

class FrappeDiagnostics:
    """Built-in diagnostic tools for Frappe applications"""
    
    def __init__(self):
        self.diagnostic_methods = self.get_diagnostic_methods()
        
    def system_health_check(self):
        """Comprehensive system health check"""
        
        health_report = {
            "timestamp": frappe.utils.now(),
            "site": frappe.local.site,
            "frappe_version": frappe.__version__,
            "checks": {}
        }
        
        # Database connectivity
        try:
            frappe.db.sql("SELECT 1")
            health_report["checks"]["database"] = {
                "status": "healthy",
                "response_time": self.measure_db_response_time()
            }
        except Exception as e:
            health_report["checks"]["database"] = {
                "status": "error",
                "error": str(e)
            }
            
        # Redis connectivity
        try:
            frappe.cache().ping()
            health_report["checks"]["redis"] = {"status": "healthy"}
        except Exception as e:
            health_report["checks"]["redis"] = {
                "status": "error", 
                "error": str(e)
            }
            
        # File system access
        try:
            test_file = frappe.get_site_path("test_write.txt")
            with open(test_file, 'w') as f:
                f.write("test")
            os.remove(test_file)
            health_report["checks"]["filesystem"] = {"status": "healthy"}
        except Exception as e:
            health_report["checks"]["filesystem"] = {
                "status": "error",
                "error": str(e)
            }
            
        # Background jobs
        try:
            queue_status = self.check_background_queues()
            health_report["checks"]["background_jobs"] = queue_status
        except Exception as e:
            health_report["checks"]["background_jobs"] = {
                "status": "error",
                "error": str(e)
            }
            
        return health_report
        
    def measure_db_response_time(self):
        """Measure database response time"""
        
        import time
        start_time = time.time()
        frappe.db.sql("SELECT COUNT(*) FROM `tabUser`")
        return round((time.time() - start_time) * 1000, 2)  # milliseconds
        
    def check_background_queues(self):
        """Check status of background job queues"""
        
        try:
            from rq import Queue
            import redis
            
            redis_conn = redis.Redis.from_url(frappe.conf.redis_queue)
            queues = ["default", "short", "long"]
            
            queue_status = {"status": "healthy", "queues": {}}
            
            for queue_name in queues:
                queue = Queue(queue_name, connection=redis_conn)
                queue_status["queues"][queue_name] = {
                    "waiting": len(queue),
                    "failed": len(queue.failed_job_registry),
                    "workers": len(queue.get_workers())
                }
                
            return queue_status
        except Exception as e:
            return {"status": "error", "error": str(e)}
            
    def performance_snapshot(self):
        """Take a performance snapshot"""
        
        import psutil
        
        snapshot = {
            "timestamp": frappe.utils.now(),
            "system": {
                "cpu_percent": psutil.cpu_percent(interval=1),
                "memory_percent": psutil.virtual_memory().percent,
                "disk_usage": psutil.disk_usage('/').percent,
                "load_average": list(psutil.getloadavg())
            },
            "database": {
                "active_connections": self.get_db_connections(),
                "slow_queries": self.get_slow_query_count(),
                "table_locks": self.get_table_locks()
            },
            "application": {
                "active_sessions": self.get_active_sessions(),
                "memory_usage": self.get_memory_usage()
            }
        }
        
        return snapshot
        
    def get_db_connections(self):
        """Get number of active database connections"""
        
        try:
            result = frappe.db.sql("SHOW STATUS LIKE 'Threads_connected'", as_dict=True)
            return int(result[0]["Value"]) if result else 0
        except:
            return "Unable to fetch"
            
    def get_slow_query_count(self):
        """Get count of slow queries"""
        
        try:
            result = frappe.db.sql("SHOW STATUS LIKE 'Slow_queries'", as_dict=True)
            return int(result[0]["Value"]) if result else 0
        except:
            return "Unable to fetch"
            
    def get_table_locks(self):
        """Get table lock status"""
        
        try:
            result = frappe.db.sql("SHOW STATUS LIKE 'Table_locks_waited'", as_dict=True)
            return int(result[0]["Value"]) if result else 0
        except:
            return "Unable to fetch"
            
    def get_active_sessions(self):
        """Get count of active user sessions"""
        
        try:
            return frappe.db.count("Sessions", {"last_updated": [">=", frappe.utils.add_to_date(frappe.utils.now(), minutes=-30)]})
        except:
            return "Unable to fetch"
            
    def get_memory_usage(self):
        """Get application memory usage"""
        
        try:
            import resource
            memory_usage = resource.getrusage(resource.RUSAGE_SELF).ru_maxrss
            # Convert to MB (Linux returns KB, macOS returns bytes)
            if sys.platform == "darwin":
                memory_usage = memory_usage / 1024 / 1024
            else:
                memory_usage = memory_usage / 1024
            return f"{memory_usage:.2f} MB"
        except:
            return "Unable to fetch"

# Command-line diagnostic tools
def run_diagnostics():
    """Run comprehensive diagnostics from command line"""
    
    # Usage: bench --site site1.local execute frappe.debug.run_diagnostics
    
    diagnostics = FrappeDiagnostics()
    
    print("=" * 50)
    print("FRAPPE SYSTEM DIAGNOSTICS")
    print("=" * 50)
    
    # System health check
    health = diagnostics.system_health_check()
    print("\n1. SYSTEM HEALTH CHECK")
    print("-" * 30)
    
    for check, result in health["checks"].items():
        status = result["status"]
        symbol = "✓" if status == "healthy" else "✗"
        print(f"{symbol} {check.title()}: {status}")
        
        if "error" in result:
            print(f"   Error: {result['error']}")
        elif "response_time" in result:
            print(f"   Response Time: {result['response_time']}ms")
            
    # Performance snapshot
    performance = diagnostics.performance_snapshot()
    print("\n2. PERFORMANCE SNAPSHOT")
    print("-" * 30)
    
    system = performance["system"]
    print(f"CPU Usage: {system['cpu_percent']}%")
    print(f"Memory Usage: {system['memory_percent']}%")
    print(f"Disk Usage: {system['disk_usage']}%")
    print(f"Load Average: {system['load_average']}")
    
    database = performance["database"]
    print(f"DB Connections: {database['active_connections']}")
    print(f"Slow Queries: {database['slow_queries']}")
    
    application = performance["application"]
    print(f"Active Sessions: {application['active_sessions']}")
    print(f"Memory Usage: {application['memory_usage']}")
    
    print("\n" + "=" * 50)
```

### Debug Mode and Logging

```python
# debugging/debug_mode.py

def enable_debug_mode():
    """Enable comprehensive debugging"""
    
    debug_settings = {
        # Enable developer mode
        "developer_mode": 1,
        
        # Enable detailed error reporting
        "log_level": "DEBUG",
        "debug_toolbar": 1,
        
        # Database query logging
        "log_queries": 1,
        "query_debug": 1,
        
        # Performance monitoring
        "monitor": 1,
        "profile": 1,
        
        # Email debugging
        "mail_debug": 1,
        "always_use_account_email_id_as_sender": 0,
        
        # Disable caching for debugging
        "disable_cache": 1,
        
        # Enable server script debugging
        "server_script_enabled": 1
    }
    
    return debug_settings

def setup_advanced_logging():
    """Set up advanced logging for troubleshooting"""
    
    import logging
    import logging.handlers
    
    # Create detailed formatter
    formatter = logging.Formatter(
        '%(asctime)s - %(name)s - %(levelname)s - %(filename)s:%(lineno)d - %(funcName)s - %(message)s'
    )
    
    # File handler with rotation
    file_handler = logging.handlers.RotatingFileHandler(
        frappe.get_site_path("logs", "debug.log"),
        maxBytes=50*1024*1024,  # 50MB
        backupCount=5
    )
    file_handler.setFormatter(formatter)
    file_handler.setLevel(logging.DEBUG)
    
    # Error-only file handler
    error_handler = logging.handlers.RotatingFileHandler(
        frappe.get_site_path("logs", "errors.log"),
        maxBytes=10*1024*1024,  # 10MB
        backupCount=3
    )
    error_handler.setFormatter(formatter)
    error_handler.setLevel(logging.ERROR)
    
    # Database query handler
    db_handler = logging.handlers.RotatingFileHandler(
        frappe.get_site_path("logs", "database.log"),
        maxBytes=100*1024*1024,  # 100MB
        backupCount=2
    )
    db_handler.setFormatter(formatter)
    
    # Configure loggers
    loggers = [
        ("frappe", logging.DEBUG),
        ("frappe.database", logging.INFO),
        ("frappe.auth", logging.INFO),
        ("frappe.email", logging.DEBUG),
        ("frappe.integrations", logging.DEBUG)
    ]
    
    for logger_name, level in loggers:
        logger = logging.getLogger(logger_name)
        logger.setLevel(level)
        logger.addHandler(file_handler)
        logger.addHandler(error_handler)
        
        if logger_name == "frappe.database":
            logger.addHandler(db_handler)

class ErrorTracker:
    """Track and analyze errors"""
    
    def __init__(self):
        self.error_patterns = self.get_common_error_patterns()
        
    def get_common_error_patterns(self):
        """Define common error patterns and solutions"""
        
        return {
            "permission_error": {
                "patterns": [
                    "PermissionError",
                    "Insufficient Permission",
                    "Not permitted"
                ],
                "solutions": [
                    "Check user roles and permissions",
                    "Verify DocPerm settings", 
                    "Check User Permissions",
                    "Verify document ownership"
                ]
            },
            
            "database_error": {
                "patterns": [
                    "DatabaseError",
                    "OperationalError",
                    "Connection refused",
                    "Too many connections"
                ],
                "solutions": [
                    "Check database server status",
                    "Verify connection settings",
                    "Check connection pool limits",
                    "Monitor database performance"
                ]
            },
            
            "validation_error": {
                "patterns": [
                    "ValidationError",
                    "MandatoryError",
                    "LinkValidationError"
                ],
                "solutions": [
                    "Check field validation rules",
                    "Verify required fields",
                    "Check link field references",
                    "Review custom validation code"
                ]
            }
        }
        
    def analyze_error_log(self, log_file_path):
        """Analyze error log for patterns"""
        
        analysis = {
            "error_summary": {},
            "recommendations": [],
            "critical_errors": []
        }
        
        try:
            with open(log_file_path, 'r') as f:
                lines = f.readlines()
                
            for line in lines:
                for error_type, config in self.error_patterns.items():
                    for pattern in config["patterns"]:
                        if pattern in line:
                            if error_type not in analysis["error_summary"]:
                                analysis["error_summary"][error_type] = 0
                            analysis["error_summary"][error_type] += 1
                            
                            # Add recommendations for this error type
                            analysis["recommendations"].extend(config["solutions"])
                            
                            # Mark as critical if it appears frequently
                            if "Critical" in line or "CRITICAL" in line:
                                analysis["critical_errors"].append({
                                    "type": error_type,
                                    "line": line.strip()
                                })
                                
        except Exception as e:
            analysis["error"] = f"Unable to analyze log: {str(e)}"
            
        return analysis
```

## Application Issues

### Common Application Errors

```python
# troubleshooting/application_issues.py

class ApplicationTroubleshooter:
    """Troubleshoot common application issues"""
    
    def diagnose_startup_issues(self):
        """Diagnose application startup problems"""
        
        startup_checks = {
            "database_connection": {
                "description": "Check database connectivity",
                "test": self.test_database_connection,
                "fixes": [
                    "Verify database server is running",
                    "Check database credentials in site_config.json",
                    "Ensure database exists and has proper permissions",
                    "Check firewall settings"
                ]
            },
            
            "redis_connection": {
                "description": "Check Redis connectivity", 
                "test": self.test_redis_connection,
                "fixes": [
                    "Verify Redis server is running",
                    "Check Redis configuration",
                    "Verify Redis password if set",
                    "Check Redis memory usage"
                ]
            },
            
            "site_configuration": {
                "description": "Validate site configuration",
                "test": self.test_site_config,
                "fixes": [
                    "Check site_config.json syntax",
                    "Verify required configuration keys",
                    "Check file permissions",
                    "Validate encryption key"
                ]
            },
            
            "app_installation": {
                "description": "Verify app installation",
                "test": self.test_app_installation,
                "fixes": [
                    "Run 'bench migrate' to update database",
                    "Check for missing app dependencies",
                    "Verify app is listed in apps.txt",
                    "Check for conflicting apps"
                ]
            }
        }
        
        results = {}
        
        for check_name, check_config in startup_checks.items():
            try:
                test_result = check_config["test"]()
                results[check_name] = {
                    "status": "pass" if test_result else "fail",
                    "result": test_result,
                    "fixes": check_config["fixes"] if not test_result else []
                }
            except Exception as e:
                results[check_name] = {
                    "status": "error",
                    "error": str(e),
                    "fixes": check_config["fixes"]
                }
                
        return results
        
    def test_database_connection(self):
        """Test database connection"""
        
        try:
            frappe.db.sql("SELECT 1")
            return True
        except Exception:
            return False
            
    def test_redis_connection(self):
        """Test Redis connection"""
        
        try:
            frappe.cache().ping()
            return True
        except Exception:
            return False
            
    def test_site_config(self):
        """Test site configuration validity"""
        
        try:
            config_path = frappe.get_site_path("site_config.json")
            with open(config_path, 'r') as f:
                config = json.load(f)
                
            # Check required keys
            required_keys = ["db_name", "db_password"]
            for key in required_keys:
                if key not in config:
                    return False
                    
            return True
        except Exception:
            return False
            
    def test_app_installation(self):
        """Test app installation status"""
        
        try:
            # Check if all apps in apps.txt are installed
            apps_path = frappe.get_site_path("apps.txt")
            if not os.path.exists(apps_path):
                return False
                
            with open(apps_path, 'r') as f:
                apps = f.read().strip().split('\n')
                
            for app in apps:
                if app and not frappe.db.exists("Module Def", {"app_name": app}):
                    return False
                    
            return True
        except Exception:
            return False
            
    def fix_common_issues(self, issue_type):
        """Apply fixes for common issues"""
        
        fixes = {
            "cache_clear": {
                "description": "Clear all caches",
                "commands": [
                    "bench --site {site} clear-cache",
                    "bench --site {site} clear-website-cache"
                ]
            },
            
            "database_migrate": {
                "description": "Run database migrations",
                "commands": [
                    "bench --site {site} migrate"
                ]
            },
            
            "rebuild_assets": {
                "description": "Rebuild static assets",
                "commands": [
                    "bench build --apps {app}",
                    "bench --site {site} clear-cache"
                ]
            },
            
            "restart_services": {
                "description": "Restart application services",
                "commands": [
                    "sudo supervisorctl restart all",
                    "sudo systemctl restart nginx"
                ]
            },
            
            "reset_permissions": {
                "description": "Reset file permissions",
                "commands": [
                    "chmod -R 755 {bench_path}/sites",
                    "chown -R {user}:{user} {bench_path}/sites"
                ]
            }
        }
        
        return fixes.get(issue_type, {})

# Error resolution guides
class ErrorResolutionGuide:
    """Specific error resolution guides"""
    
    def __init__(self):
        self.error_solutions = self.build_error_database()
        
    def build_error_database(self):
        """Build comprehensive error solution database"""
        
        return {
            # Permission Errors
            "PermissionError": {
                "description": "User lacks permission to perform action",
                "common_causes": [
                    "User doesn't have required role",
                    "DocPerm not configured correctly",
                    "User Permission restricts access",
                    "Document is submitted and user can't modify"
                ],
                "resolution_steps": [
                    "Check user's roles in User document",
                    "Verify DocPerm settings for the DocType",
                    "Check User Permission restrictions",
                    "Use 'ignore_permissions=True' in code if appropriate",
                    "Check if document can be modified in current state"
                ],
                "prevention": [
                    "Set up proper role hierarchy",
                    "Test permissions thoroughly",
                    "Document permission requirements"
                ]
            },
            
            # Database Errors
            "DatabaseError: Connection refused": {
                "description": "Cannot connect to database server",
                "common_causes": [
                    "Database server not running",
                    "Wrong connection parameters",
                    "Network connectivity issues",
                    "Firewall blocking connection"
                ],
                "resolution_steps": [
                    "Check if MariaDB/MySQL is running: systemctl status mysql",
                    "Verify database credentials in site_config.json",
                    "Test connection: mysql -u user -p -h host",
                    "Check firewall rules: ufw status",
                    "Review database server logs"
                ],
                "prevention": [
                    "Set up database monitoring",
                    "Configure automatic database restarts",
                    "Regular database health checks"
                ]
            },
            
            # Import/Export Errors
            "DataError: Incorrect datetime value": {
                "description": "Invalid datetime format in data",
                "common_causes": [
                    "Incorrect date/time format in import data",
                    "Timezone conversion issues",
                    "Empty or null datetime values"
                ],
                "resolution_steps": [
                    "Check date format in import data (YYYY-MM-DD HH:MM:SS)",
                    "Use frappe.utils.getdate() for date conversion",
                    "Handle empty datetime values in code",
                    "Set proper timezone in system settings"
                ],
                "prevention": [
                    "Validate date formats before import",
                    "Use consistent datetime handling",
                    "Add date format validation in forms"
                ]
            },
            
            # Session Errors
            "SessionExpired": {
                "description": "User session has expired",
                "common_causes": [
                    "Session timeout reached",
                    "Server restart cleared sessions",
                    "Redis server unavailable",
                    "Session data corruption"
                ],
                "resolution_steps": [
                    "Check Redis server status",
                    "Verify session configuration",
                    "Clear browser cookies",
                    "Restart application server",
                    "Check system time synchronization"
                ],
                "prevention": [
                    "Configure appropriate session timeout",
                    "Set up Redis monitoring",
                    "Use persistent session storage"
                ]
            },
            
            # File Upload Errors
            "File size exceeds maximum allowed": {
                "description": "Uploaded file is too large",
                "common_causes": [
                    "File larger than max_file_size setting",
                    "Nginx client_max_body_size limit",
                    "PHP upload limits (if using)",
                    "Disk space issues"
                ],
                "resolution_steps": [
                    "Increase max_file_size in site_config.json",
                    "Update nginx client_max_body_size",
                    "Check available disk space",
                    "Optimize file before upload",
                    "Use cloud storage for large files"
                ],
                "prevention": [
                    "Set appropriate file size limits",
                    "Implement file compression",
                    "Use progressive upload for large files"
                ]
            }
        }
        
    def get_solution(self, error_message):
        """Get solution for specific error"""
        
        for error_pattern, solution in self.error_solutions.items():
            if error_pattern.lower() in error_message.lower():
                return solution
                
        return {
            "description": "Unknown error",
            "resolution_steps": [
                "Check application logs for details",
                "Search Frappe forum for similar issues", 
                "Check GitHub issues",
                "Enable debug mode for more information"
            ]
        }
```

## Database Problems

### Database Troubleshooting

```python
# troubleshooting/database_issues.py

class DatabaseTroubleshooter:
    """Troubleshoot database-related issues"""
    
    def __init__(self):
        self.db = frappe.db
        
    def diagnose_connection_issues(self):
        """Diagnose database connection problems"""
        
        connection_tests = {
            "basic_connectivity": self.test_basic_connection,
            "authentication": self.test_authentication,
            "database_existence": self.test_database_exists,
            "table_access": self.test_table_access,
            "connection_limits": self.test_connection_limits
        }
        
        results = {}
        for test_name, test_func in connection_tests.items():
            try:
                results[test_name] = {
                    "status": "pass" if test_func() else "fail",
                    "timestamp": frappe.utils.now()
                }
            except Exception as e:
                results[test_name] = {
                    "status": "error",
                    "error": str(e),
                    "timestamp": frappe.utils.now()
                }
                
        return results
        
    def test_basic_connection(self):
        """Test basic database connection"""
        try:
            self.db.sql("SELECT 1")
            return True
        except Exception:
            return False
            
    def test_authentication(self):
        """Test database authentication"""
        try:
            self.db.sql("SELECT USER(), DATABASE()")
            return True
        except Exception:
            return False
            
    def test_database_exists(self):
        """Test if database exists"""
        try:
            result = self.db.sql(f"SHOW DATABASES LIKE '{frappe.conf.db_name}'")
            return len(result) > 0
        except Exception:
            return False
            
    def test_table_access(self):
        """Test table access permissions"""
        try:
            self.db.sql("SELECT COUNT(*) FROM `tabUser` LIMIT 1")
            return True
        except Exception:
            return False
            
    def test_connection_limits(self):
        """Check connection limits"""
        try:
            result = self.db.sql("SHOW STATUS LIKE 'Threads_connected'", as_dict=True)
            connected = int(result[0]["Value"])
            
            result = self.db.sql("SHOW VARIABLES LIKE 'max_connections'", as_dict=True)
            max_connections = int(result[0]["Value"])
            
            # Return True if we're not near the limit (less than 80%)
            return connected < (max_connections * 0.8)
        except Exception:
            return False
            
    def analyze_slow_queries(self):
        """Analyze slow queries"""
        
        slow_query_analysis = {
            "enabled": False,
            "slow_queries": [],
            "recommendations": []
        }
        
        try:
            # Check if slow query log is enabled
            result = self.db.sql("SHOW VARIABLES LIKE 'slow_query_log'", as_dict=True)
            if result and result[0]["Value"] == "ON":
                slow_query_analysis["enabled"] = True
                
                # Get slow query statistics
                stats = self.db.sql("""
                    SHOW STATUS LIKE 'Slow_queries'
                """, as_dict=True)
                
                if stats:
                    slow_count = int(stats[0]["Value"])
                    slow_query_analysis["total_slow_queries"] = slow_count
                    
                    if slow_count > 0:
                        slow_query_analysis["recommendations"] = [
                            "Review and optimize slow queries",
                            "Add appropriate database indexes",
                            "Check for missing WHERE clauses",
                            "Consider query restructuring"
                        ]
                        
            # Get queries from performance schema if available
            try:
                slow_queries = self.db.sql("""
                    SELECT 
                        sql_text,
                        exec_count,
                        avg_timer_wait/1000000000 as avg_time_sec,
                        sum_rows_examined/exec_count as avg_rows_examined
                    FROM performance_schema.statements_summary_by_digest
                    WHERE avg_timer_wait > 1000000000
                    ORDER BY avg_timer_wait DESC
                    LIMIT 10
                """, as_dict=True)
                
                slow_query_analysis["slow_queries"] = slow_queries
                
            except Exception:
                # Performance schema might not be available
                pass
                
        except Exception as e:
            slow_query_analysis["error"] = str(e)
            
        return slow_query_analysis
        
    def check_database_integrity(self):
        """Check database integrity"""
        
        integrity_checks = {
            "table_status": [],
            "orphaned_records": [],
            "foreign_key_violations": [],
            "recommendations": []
        }
        
        try:
            # Check table status
            tables = self.db.sql("SHOW TABLES", as_list=True)
            
            for table in tables:
                table_name = table[0]
                status = self.db.sql(f"CHECK TABLE `{table_name}`", as_dict=True)
                
                if status and status[0]["Msg_text"] != "OK":
                    integrity_checks["table_status"].append({
                        "table": table_name,
                        "status": status[0]["Msg_text"]
                    })
                    
            # Check for orphaned records in common tables
            orphan_checks = [
                {
                    "table": "tabDocShare",
                    "query": """
                        SELECT COUNT(*) as count
                        FROM `tabDocShare` ds
                        LEFT JOIN `tabUser` u ON ds.user = u.name
                        WHERE u.name IS NULL
                    """
                },
                {
                    "table": "tabActivity Log", 
                    "query": """
                        SELECT COUNT(*) as count
                        FROM `tabActivity Log` al
                        LEFT JOIN `tabUser` u ON al.user = u.name
                        WHERE u.name IS NULL AND al.user IS NOT NULL
                    """
                }
            ]
            
            for check in orphan_checks:
                try:
                    result = self.db.sql(check["query"], as_dict=True)
                    if result and result[0]["count"] > 0:
                        integrity_checks["orphaned_records"].append({
                            "table": check["table"],
                            "count": result[0]["count"]
                        })
                except Exception:
                    continue
                    
            # Generate recommendations
            if integrity_checks["table_status"]:
                integrity_checks["recommendations"].append("Run REPAIR TABLE on corrupted tables")
                
            if integrity_checks["orphaned_records"]:
                integrity_checks["recommendations"].append("Clean up orphaned records")
                
        except Exception as e:
            integrity_checks["error"] = str(e)
            
        return integrity_checks
        
    def optimize_database_performance(self):
        """Provide database performance optimization recommendations"""
        
        optimization_report = {
            "current_settings": {},
            "recommendations": [],
            "queries_to_run": []
        }
        
        try:
            # Get current key settings
            settings_to_check = [
                "innodb_buffer_pool_size",
                "max_connections",
                "query_cache_size",
                "table_open_cache",
                "innodb_log_file_size"
            ]
            
            for setting in settings_to_check:
                result = self.db.sql(f"SHOW VARIABLES LIKE '{setting}'", as_dict=True)
                if result:
                    optimization_report["current_settings"][setting] = result[0]["Value"]
                    
            # Get system memory
            try:
                import psutil
                total_memory_gb = psutil.virtual_memory().total / (1024**3)
                
                # Recommendations based on memory
                innodb_buffer_pool = optimization_report["current_settings"].get("innodb_buffer_pool_size", "0")
                innodb_pool_gb = int(innodb_buffer_pool) / (1024**3) if innodb_buffer_pool.isdigit() else 0
                
                recommended_pool_size = int(total_memory_gb * 0.7)
                
                if innodb_pool_gb < recommended_pool_size:
                    optimization_report["recommendations"].append(
                        f"Increase innodb_buffer_pool_size to {recommended_pool_size}G "
                        f"(currently {innodb_pool_gb:.1f}G)"
                    )
                    
            except ImportError:
                pass
                
            # Check for tables without indexes
            tables_without_indexes = self.db.sql("""
                SELECT table_name
                FROM information_schema.tables t
                LEFT JOIN information_schema.statistics s 
                    ON t.table_name = s.table_name 
                    AND t.table_schema = s.table_schema
                WHERE t.table_schema = %s 
                    AND t.table_type = 'BASE TABLE'
                    AND s.table_name IS NULL
                    AND t.table_name LIKE 'tab%'
            """, (frappe.conf.db_name,))
            
            if tables_without_indexes:
                optimization_report["recommendations"].append(
                    f"Consider adding indexes to tables: {', '.join([t[0] for t in tables_without_indexes])}"
                )
                
            # Optimization queries
            optimization_report["queries_to_run"] = [
                "ANALYZE TABLE `tabUser`",
                "OPTIMIZE TABLE `tabActivity Log`",
                "ANALYZE TABLE `tabDocType`",
                "CHECK TABLE `tabSingles`"
            ]
            
        except Exception as e:
            optimization_report["error"] = str(e)
            
        return optimization_report
```

## Performance Issues

### Performance Diagnostics

```python
# troubleshooting/performance_issues.py

class PerformanceTroubleshooter:
    """Diagnose and resolve performance issues"""
    
    def __init__(self):
        self.performance_thresholds = {
            "response_time": 3.0,  # seconds
            "cpu_usage": 80,       # percent
            "memory_usage": 85,    # percent
            "disk_usage": 90,      # percent
            "db_connections": 150   # count
        }
        
    def comprehensive_performance_check(self):
        """Run comprehensive performance diagnostics"""
        
        performance_report = {
            "timestamp": frappe.utils.now(),
            "overall_status": "unknown",
            "checks": {},
            "recommendations": [],
            "critical_issues": []
        }
        
        # System performance
        system_perf = self.check_system_performance()
        performance_report["checks"]["system"] = system_perf
        
        # Application performance
        app_perf = self.check_application_performance()
        performance_report["checks"]["application"] = app_perf
        
        # Database performance
        db_perf = self.check_database_performance()
        performance_report["checks"]["database"] = db_perf
        
        # Cache performance
        cache_perf = self.check_cache_performance()
        performance_report["checks"]["cache"] = cache_perf
        
        # Determine overall status
        critical_count = sum([
            1 for check in performance_report["checks"].values()
            if check.get("status") == "critical"
        ])
        
        if critical_count > 0:
            performance_report["overall_status"] = "critical"
        elif critical_count == 0:
            warning_count = sum([
                1 for check in performance_report["checks"].values()
                if check.get("status") == "warning"
            ])
            performance_report["overall_status"] = "warning" if warning_count > 0 else "healthy"
            
        # Generate recommendations
        performance_report["recommendations"] = self.generate_performance_recommendations(
            performance_report["checks"]
        )
        
        return performance_report
        
    def check_system_performance(self):
        """Check system-level performance"""
        
        try:
            import psutil
            
            # CPU usage
            cpu_percent = psutil.cpu_percent(interval=1)
            
            # Memory usage
            memory = psutil.virtual_memory()
            memory_percent = memory.percent
            
            # Disk usage
            disk = psutil.disk_usage('/')
            disk_percent = disk.percent
            
            # Load average
            load_avg = psutil.getloadavg()
            
            system_status = "healthy"
            issues = []
            
            if cpu_percent > self.performance_thresholds["cpu_usage"]:
                system_status = "critical"
                issues.append(f"High CPU usage: {cpu_percent}%")
                
            if memory_percent > self.performance_thresholds["memory_usage"]:
                system_status = "critical"
                issues.append(f"High memory usage: {memory_percent}%")
                
            if disk_percent > self.performance_thresholds["disk_usage"]:
                system_status = "warning"
                issues.append(f"High disk usage: {disk_percent}%")
                
            return {
                "status": system_status,
                "metrics": {
                    "cpu_percent": cpu_percent,
                    "memory_percent": memory_percent,
                    "disk_percent": disk_percent,
                    "load_average": load_avg
                },
                "issues": issues
            }
            
        except ImportError:
            return {
                "status": "error",
                "error": "psutil not available for system monitoring"
            }
            
    def check_application_performance(self):
        """Check application-level performance"""
        
        app_metrics = {
            "status": "healthy",
            "metrics": {},
            "issues": []
        }
        
        try:
            # Active sessions
            active_sessions = frappe.db.count("Sessions", {
                "last_updated": [">=", frappe.utils.add_to_date(frappe.utils.now(), minutes=-30)]
            })
            app_metrics["metrics"]["active_sessions"] = active_sessions
            
            # Background job queue length
            try:
                from rq import Queue
                import redis
                
                redis_conn = redis.Redis.from_url(frappe.conf.redis_queue)
                default_queue = Queue("default", connection=redis_conn)
                queue_length = len(default_queue)
                
                app_metrics["metrics"]["queue_length"] = queue_length
                
                if queue_length > 50:
                    app_metrics["status"] = "warning"
                    app_metrics["issues"].append(f"High background job queue: {queue_length}")
                    
            except Exception:
                app_metrics["metrics"]["queue_length"] = "unavailable"
                
            # Recent error count
            error_count = frappe.db.count("Error Log", {
                "creation": [">=", frappe.utils.add_to_date(frappe.utils.now(), hours=-1)]
            })
            app_metrics["metrics"]["recent_errors"] = error_count
            
            if error_count > 10:
                app_metrics["status"] = "warning"
                app_metrics["issues"].append(f"High error rate: {error_count} errors in last hour")
                
        except Exception as e:
            app_metrics["status"] = "error"
            app_metrics["error"] = str(e)
            
        return app_metrics
        
    def check_database_performance(self):
        """Check database performance"""
        
        db_metrics = {
            "status": "healthy",
            "metrics": {},
            "issues": []
        }
        
        try:
            # Connection count
            result = frappe.db.sql("SHOW STATUS LIKE 'Threads_connected'", as_dict=True)
            connections = int(result[0]["Value"]) if result else 0
            db_metrics["metrics"]["connections"] = connections
            
            if connections > self.performance_thresholds["db_connections"]:
                db_metrics["status"] = "critical"
                db_metrics["issues"].append(f"High database connections: {connections}")
                
            # Slow queries
            result = frappe.db.sql("SHOW STATUS LIKE 'Slow_queries'", as_dict=True)
            slow_queries = int(result[0]["Value"]) if result else 0
            db_metrics["metrics"]["slow_queries"] = slow_queries
            
            # Query response time test
            import time
            start_time = time.time()
            frappe.db.sql("SELECT COUNT(*) FROM `tabUser`")
            query_time = (time.time() - start_time) * 1000
            
            db_metrics["metrics"]["sample_query_time_ms"] = round(query_time, 2)
            
            if query_time > 1000:  # 1 second
                db_metrics["status"] = "warning"
                db_metrics["issues"].append(f"Slow query response: {query_time:.0f}ms")
                
            # Table lock waits
            result = frappe.db.sql("SHOW STATUS LIKE 'Table_locks_waited'", as_dict=True)
            lock_waits = int(result[0]["Value"]) if result else 0
            db_metrics["metrics"]["table_lock_waits"] = lock_waits
            
        except Exception as e:
            db_metrics["status"] = "error"
            db_metrics["error"] = str(e)
            
        return db_metrics
        
    def check_cache_performance(self):
        """Check cache performance"""
        
        cache_metrics = {
            "status": "healthy",
            "metrics": {},
            "issues": []
        }
        
        try:
            # Redis info
            redis_info = frappe.cache().connection_pool.connection().info()
            
            cache_metrics["metrics"] = {
                "used_memory_mb": round(redis_info.get("used_memory", 0) / 1024 / 1024, 2),
                "connected_clients": redis_info.get("connected_clients", 0),
                "keyspace_hits": redis_info.get("keyspace_hits", 0),
                "keyspace_misses": redis_info.get("keyspace_misses", 0)
            }
            
            # Calculate hit rate
            hits = cache_metrics["metrics"]["keyspace_hits"]
            misses = cache_metrics["metrics"]["keyspace_misses"]
            
            if hits + misses > 0:
                hit_rate = (hits / (hits + misses)) * 100
                cache_metrics["metrics"]["hit_rate_percent"] = round(hit_rate, 2)
                
                if hit_rate < 80:
                    cache_metrics["status"] = "warning"
                    cache_metrics["issues"].append(f"Low cache hit rate: {hit_rate:.1f}%")
                    
            # Memory usage check
            if redis_info.get("used_memory_rss", 0) > 1024 * 1024 * 1024:  # 1GB
                cache_metrics["status"] = "warning"
                cache_metrics["issues"].append("High Redis memory usage")
                
        except Exception as e:
            cache_metrics["status"] = "error"
            cache_metrics["error"] = str(e)
            
        return cache_metrics
        
    def generate_performance_recommendations(self, checks):
        """Generate performance optimization recommendations"""
        
        recommendations = []
        
        # System recommendations
        system_check = checks.get("system", {})
        if system_check.get("status") in ["warning", "critical"]:
            for issue in system_check.get("issues", []):
                if "CPU" in issue:
                    recommendations.extend([
                        "Consider upgrading CPU or reducing CPU-intensive operations",
                        "Optimize application code for better CPU efficiency",
                        "Scale horizontally by adding more application servers"
                    ])
                elif "memory" in issue:
                    recommendations.extend([
                        "Add more RAM to the server",
                        "Optimize memory usage in applications",
                        "Consider using swap file as temporary solution"
                    ])
                elif "disk" in issue:
                    recommendations.extend([
                        "Clean up log files and temporary data",
                        "Archive old data to separate storage",
                        "Consider adding more disk space"
                    ])
                    
        # Database recommendations
        db_check = checks.get("database", {})
        if db_check.get("status") in ["warning", "critical"]:
            recommendations.extend([
                "Optimize database queries and add appropriate indexes",
                "Consider connection pooling to manage database connections",
                "Review and optimize database configuration",
                "Consider read replicas for read-heavy workloads"
            ])
            
        # Cache recommendations
        cache_check = checks.get("cache", {})
        if cache_check.get("status") in ["warning", "critical"]:
            recommendations.extend([
                "Optimize cache usage and implement cache warming",
                "Review cache expiration policies",
                "Consider increasing Redis memory allocation",
                "Implement cache partitioning for better performance"
            ])
            
        return list(set(recommendations))  # Remove duplicates

class SlowQueryAnalyzer:
    """Analyze and optimize slow queries"""
    
    def analyze_query_performance(self, query):
        """Analyze specific query performance"""
        
        analysis = {
            "query": query,
            "execution_plan": None,
            "recommendations": [],
            "estimated_improvement": None
        }
        
        try:
            # Get execution plan
            explain_result = frappe.db.sql(f"EXPLAIN {query}", as_dict=True)
            analysis["execution_plan"] = explain_result
            
            # Analyze execution plan
            for row in explain_result:
                # Check for full table scans
                if row.get("type") == "ALL":
                    analysis["recommendations"].append(
                        f"Full table scan on {row.get('table')} - consider adding index"
                    )
                    
                # Check for filesort
                if "Using filesort" in str(row.get("Extra", "")):
                    analysis["recommendations"].append(
                        "Query uses filesort - consider adding index for ORDER BY"
                    )
                    
                # Check for temporary tables
                if "Using temporary" in str(row.get("Extra", "")):
                    analysis["recommendations"].append(
                        "Query uses temporary table - consider query optimization"
                    )
                    
                # Check for high row examination
                if row.get("rows", 0) > 1000:
                    analysis["recommendations"].append(
                        f"High row examination ({row.get('rows')}) - consider more selective WHERE clause"
                    )
                    
        except Exception as e:
            analysis["error"] = str(e)
            
        return analysis
```

## Authentication and Permissions

### Authentication Issues

```python
# troubleshooting/auth_issues.py

class AuthTroubleshooter:
    """Troubleshoot authentication and permission issues"""
    
    def diagnose_login_issues(self, user_email=None):
        """Diagnose login problems"""
        
        diagnosis = {
            "user_status": None,
            "session_status": None,
            "permission_status": None,
            "issues": [],
            "solutions": []
        }
        
        if user_email:
            # Check user account status
            user_check = self.check_user_account(user_email)
            diagnosis["user_status"] = user_check
            
            if not user_check["exists"]:
                diagnosis["issues"].append("User account does not exist")
                diagnosis["solutions"].append("Create user account or check email spelling")
            elif not user_check["enabled"]:
                diagnosis["issues"].append("User account is disabled")
                diagnosis["solutions"].append("Enable user account in User master")
            elif user_check["login_blocked"]:
                diagnosis["issues"].append("User login is blocked due to failed attempts")
                diagnosis["solutions"].append("Reset login attempts or wait for timeout")
                
        # Check session configuration
        session_check = self.check_session_config()
        diagnosis["session_status"] = session_check
        
        if session_check.get("issues"):
            diagnosis["issues"].extend(session_check["issues"])
            diagnosis["solutions"].extend(session_check["solutions"])
            
        return diagnosis
        
    def check_user_account(self, email):
        """Check user account status"""
        
        user_status = {
            "exists": False,
            "enabled": False,
            "login_blocked": False,
            "roles": [],
            "last_login": None,
            "failed_logins": 0
        }
        
        try:
            user = frappe.get_doc("User", email)
            user_status["exists"] = True
            user_status["enabled"] = user.enabled
            user_status["roles"] = [r.role for r in user.roles]
            user_status["last_login"] = user.last_login
            
            # Check for login blocks
            if user.login_after and frappe.utils.getdate() < frappe.utils.getdate(user.login_after):
                user_status["login_blocked"] = True
                
        except frappe.DoesNotExistError:
            pass
        except Exception as e:
            user_status["error"] = str(e)
            
        return user_status
        
    def check_session_config(self):
        """Check session configuration"""
        
        session_check = {
            "redis_available": False,
            "session_expiry": None,
            "issues": [],
            "solutions": []
        }
        
        try:
            # Check Redis connectivity
            frappe.cache().ping()
            session_check["redis_available"] = True
            
            # Check session expiry setting
            session_expiry = frappe.db.get_single_value("System Settings", "session_expiry")
            session_check["session_expiry"] = session_expiry
            
            if not session_expiry:
                session_check["issues"].append("Session expiry not configured")
                session_check["solutions"].append("Set session expiry in System Settings")
                
        except Exception as e:
            session_check["issues"].append(f"Redis connection failed: {str(e)}")
            session_check["solutions"].extend([
                "Check Redis server status",
                "Verify Redis configuration",
                "Check network connectivity to Redis"
            ])
            
        return session_check
        
    def diagnose_permission_issues(self, user_email, doctype, docname=None, permission_type="read"):
        """Diagnose permission issues"""
        
        diagnosis = {
            "user_roles": [],
            "doctype_permissions": [],
            "user_permissions": [],
            "share_permissions": [],
            "has_permission": False,
            "blocking_factors": [],
            "solutions": []
        }
        
        try:
            # Get user roles
            user_roles = frappe.get_roles(user_email)
            diagnosis["user_roles"] = user_roles
            
            # Get DocType permissions
            doctype_perms = frappe.get_all("DocPerm",
                filters={"parent": doctype, "role": ["in", user_roles]},
                fields=["role", "read", "write", "create", "delete", "submit", "cancel", "permlevel"]
            )
            diagnosis["doctype_permissions"] = doctype_perms
            
            # Check if user has any permissions for this DocType
            has_role_permission = any([
                perm.get(permission_type, 0) for perm in doctype_perms
            ])
            
            if not has_role_permission:
                diagnosis["blocking_factors"].append(f"No {permission_type} permission for any user roles")
                diagnosis["solutions"].append(f"Add {permission_type} permission for one of user's roles: {', '.join(user_roles)}")
                
            # Get User Permissions
            user_perms = frappe.get_all("User Permission",
                filters={"user": user_email},
                fields=["allow", "for_value", "applicable_for"]
            )
            diagnosis["user_permissions"] = user_perms
            
            # Check document-specific permissions
            if docname:
                # Check share permissions
                share_perms = frappe.get_all("DocShare",
                    filters={
                        "share_doctype": doctype,
                        "share_name": docname,
                        "user": user_email
                    },
                    fields=["read", "write", "share"]
                )
                diagnosis["share_permissions"] = share_perms
                
                # Test actual permission
                diagnosis["has_permission"] = frappe.has_permission(
                    doctype, permission_type, docname, user=user_email
                )
                
                if not diagnosis["has_permission"]:
                    # Try to identify why permission failed
                    if user_perms:
                        applicable_perms = [up for up in user_perms if up.get("applicable_for") == doctype]
                        if applicable_perms:
                            diagnosis["blocking_factors"].append("User Permission restrictions may be blocking access")
                            diagnosis["solutions"].append("Check User Permission settings for this user")
                            
        except Exception as e:
            diagnosis["error"] = str(e)
            
        return diagnosis
        
    def fix_permission_issues(self, issue_type, **kwargs):
        """Apply fixes for common permission issues"""
        
        fixes = {
            "add_role_permission": {
                "description": "Add role permission for DocType",
                "action": self.add_role_permission
            },
            "enable_user_account": {
                "description": "Enable disabled user account",
                "action": self.enable_user_account
            },
            "reset_login_attempts": {
                "description": "Reset failed login attempts",
                "action": self.reset_login_attempts
            },
            "share_document": {
                "description": "Share document with specific user",
                "action": self.share_document
            }
        }
        
        fix_config = fixes.get(issue_type)
        if fix_config:
            return fix_config["action"](**kwargs)
        else:
            return {"success": False, "message": f"Unknown fix type: {issue_type}"}
            
    def add_role_permission(self, role, doctype, permission_type="read"):
        """Add permission for role on DocType"""
        
        try:
            # Check if permission already exists
            existing = frappe.db.exists("DocPerm", {
                "parent": doctype,
                "role": role,
                "permlevel": 0
            })
            
            if existing:
                # Update existing permission
                perm_doc = frappe.get_doc("DocPerm", existing)
                perm_doc.set(permission_type, 1)
                perm_doc.save()
            else:
                # Create new permission
                perm_doc = frappe.get_doc({
                    "doctype": "DocPerm",
                    "parent": doctype,
                    "parenttype": "DocType",
                    "parentfield": "permissions",
                    "role": role,
                    "permlevel": 0,
                    permission_type: 1
                })
                perm_doc.insert(ignore_permissions=True)
                
            # Clear cache
            frappe.clear_cache(doctype=doctype)
            
            return {
                "success": True,
                "message": f"Added {permission_type} permission for {role} on {doctype}"
            }
            
        except Exception as e:
            return {"success": False, "message": str(e)}
            
    def enable_user_account(self, user_email):
        """Enable disabled user account"""
        
        try:
            user = frappe.get_doc("User", user_email)
            user.enabled = 1
            user.save(ignore_permissions=True)
            
            return {
                "success": True,
                "message": f"Enabled user account: {user_email}"
            }
            
        except Exception as e:
            return {"success": False, "message": str(e)}
            
    def reset_login_attempts(self, user_email):
        """Reset failed login attempts"""
        
        try:
            # Clear login attempts from cache
            frappe.cache().delete_key(f"failed_logins:{user_email}")
            
            # Clear any login restrictions
            user = frappe.get_doc("User", user_email)
            if user.login_after:
                user.login_after = None
                user.save(ignore_permissions=True)
                
            return {
                "success": True,
                "message": f"Reset login attempts for: {user_email}"
            }
            
        except Exception as e:
            return {"success": False, "message": str(e)}
            
    def share_document(self, doctype, docname, user_email, permissions=None):
        """Share document with specific user"""
        
        try:
            import frappe.share
            
            default_permissions = {"read": 1, "write": 0, "share": 0}
            permissions = permissions or default_permissions
            
            frappe.share.add(
                doctype=doctype,
                name=docname,
                user=user_email,
                **permissions
            )
            
            return {
                "success": True,
                "message": f"Shared {doctype} {docname} with {user_email}"
            }
            
        except Exception as e:
            return {"success": False, "message": str(e)}
```

## System Administration

### System Health Monitoring

```bash
#!/bin/bash
# system_health_monitor.sh

# Comprehensive system health monitoring script
monitor_system_health() {
    echo "=== SYSTEM HEALTH MONITOR ==="
    echo "Timestamp: $(date)"
    echo "================================"
    
    # Check system resources
    echo "1. SYSTEM RESOURCES"
    echo "   CPU Usage: $(top -bn1 | grep "Cpu(s)" | awk '{print $2}' | cut -d'%' -f1)%"
    echo "   Memory Usage: $(free | grep Mem | awk '{printf("%.2f%"), ($3/$2) * 100.0}')"
    echo "   Disk Usage: $(df -h / | awk 'NR==2{printf "%s", $5}')"
    echo "   Load Average: $(uptime | awk -F'load average:' '{print $2}')"
    echo ""
    
    # Check services
    echo "2. CRITICAL SERVICES"
    services=("nginx" "mysql" "redis-server" "supervisor")
    
    for service in "${services[@]}"; do
        if systemctl is-active --quiet $service; then
            echo "   ✓ $service: Running"
        else
            echo "   ✗ $service: Stopped"
        fi
    done
    echo ""
    
    # Check network connectivity
    echo "3. NETWORK CONNECTIVITY"
    if ping -c 1 google.com &> /dev/null; then
        echo "   ✓ Internet: Connected"
    else
        echo "   ✗ Internet: Disconnected"
    fi
    
    # Check local services
    if curl -s http://localhost > /dev/null; then
        echo "   ✓ Web Server: Responsive"
    else
        echo "   ✗ Web Server: Not responding"
    fi
    echo ""
    
    # Check log file sizes
    echo "4. LOG FILE SIZES"
    log_dirs=("/var/log/nginx" "/home/frappe/frappe-bench/logs")
    
    for log_dir in "${log_dirs[@]}"; do
        if [ -d "$log_dir" ]; then
            size=$(du -sh "$log_dir" 2>/dev/null | cut -f1)
            echo "   $log_dir: $size"
        fi
    done
    echo ""
    
    # Check disk space
    echo "5. DISK SPACE WARNINGS"
    df -h | awk '$5 > 80 {print "   ⚠️  " $1 " is " $5 " full"}'
    echo ""
    
    # Check for recent errors
    echo "6. RECENT ERROR SUMMARY"
    error_count=$(journalctl --since "1 hour ago" --priority=err | wc -l)
    echo "   Errors in last hour: $error_count"
    
    if [ "$error_count" -gt 0 ]; then
        echo "   Recent errors:"
        journalctl --since "1 hour ago" --priority=err --no-pager | tail -5
    fi
    echo ""
    
    # Generate recommendations
    echo "7. RECOMMENDATIONS"
    
    # High CPU usage
    cpu_usage=$(top -bn1 | grep "Cpu(s)" | awk '{print $2}' | cut -d'%' -f1 | cut -d' ' -f1)
    if (( $(echo "$cpu_usage > 80" | bc -l) )); then
        echo "   - High CPU usage detected. Consider investigating running processes."
    fi
    
    # High memory usage
    memory_usage=$(free | grep Mem | awk '{printf("%.0f"), ($3/$2) * 100.0}')
    if [ "$memory_usage" -gt 85 ]; then
        echo "   - High memory usage detected. Consider restarting services or adding RAM."
    fi
    
    # Large log files
    large_logs=$(find /var/log -name "*.log" -size +100M 2>/dev/null)
    if [ ! -z "$large_logs" ]; then
        echo "   - Large log files found. Consider log rotation:"
        echo "$large_logs" | sed 's/^/     /'
    fi
    
    echo "================================"
}

# Automated cleanup tasks
cleanup_system() {
    echo "=== SYSTEM CLEANUP ==="
    
    # Clear old log files
    echo "1. Cleaning old log files..."
    find /var/log -name "*.log" -mtime +30 -exec rm {} \;
    find /home/frappe/frappe-bench/logs -name "*.log" -mtime +30 -exec rm {} \;
    
    # Clear temporary files
    echo "2. Cleaning temporary files..."
    rm -rf /tmp/*
    
    # Clear package cache
    echo "3. Cleaning package cache..."
    apt-get clean
    
    # Clear frappe cache
    echo "4. Clearing application cache..."
    cd /home/frappe/frappe-bench
    bench --site all clear-cache
    
    echo "Cleanup completed!"
}

# Database maintenance
maintain_database() {
    echo "=== DATABASE MAINTENANCE ==="
    
    # Get database name from site config
    DB_NAME=$(python3 -c "
import json
with open('/home/frappe/frappe-bench/sites/site1.local/site_config.json') as f:
    config = json.load(f)
    print(config['db_name'])
")
    
    echo "1. Analyzing tables..."
    mysql -u root -p$MYSQL_ROOT_PASSWORD -e "
        USE $DB_NAME;
        ANALYZE TABLE \`tabUser\`;
        ANALYZE TABLE \`tabDocType\`;
        ANALYZE TABLE \`tabActivity Log\`;
    "
    
    echo "2. Optimizing tables..."
    mysql -u root -p$MYSQL_ROOT_PASSWORD -e "
        USE $DB_NAME;
        OPTIMIZE TABLE \`tabActivity Log\`;
        OPTIMIZE TABLE \`tabError Log\`;
    "
    
    echo "3. Checking table integrity..."
    mysql -u root -p$MYSQL_ROOT_PASSWORD -e "
        USE $DB_NAME;
        CHECK TABLE \`tabUser\`;
        CHECK TABLE \`tabDocType\`;
    "
    
    echo "Database maintenance completed!"
}

# Performance optimization
optimize_performance() {
    echo "=== PERFORMANCE OPTIMIZATION ==="
    
    # Restart services for fresh start
    echo "1. Restarting critical services..."
    systemctl restart nginx
    systemctl restart redis-server
    supervisorctl restart all
    
    # Clear all caches
    echo "2. Clearing all caches..."
    cd /home/frappe/frappe-bench
    bench --site all clear-cache
    bench --site all clear-website-cache
    
    # Restart Redis to clear memory
    echo "3. Optimizing Redis..."
    redis-cli FLUSHDB
    systemctl restart redis-server
    
    echo "Performance optimization completed!"
}

# Main menu
main() {
    case "$1" in
        "monitor")
            monitor_system_health
            ;;
        "cleanup")
            cleanup_system
            ;;
        "database")
            maintain_database
            ;;
        "optimize")
            optimize_performance
            ;;
        *)
            echo "Usage: $0 {monitor|cleanup|database|optimize}"
            echo ""
            echo "Commands:"
            echo "  monitor  - Run system health check"
            echo "  cleanup  - Clean temporary files and logs"
            echo "  database - Perform database maintenance"
            echo "  optimize - Optimize system performance"
            exit 1
            ;;
    esac
}

main "$@"
```

## Recovery Procedures

### Disaster Recovery

```python
# recovery/disaster_recovery.py

class DisasterRecoveryManager:
    """Manage disaster recovery procedures"""
    
    def __init__(self):
        self.recovery_procedures = self.define_recovery_procedures()
        
    def define_recovery_procedures(self):
        """Define different disaster recovery procedures"""
        
        return {
            "database_corruption": {
                "severity": "critical",
                "estimated_downtime": "2-4 hours",
                "steps": [
                    "Stop all application services",
                    "Assess database corruption extent",
                    "Restore from latest backup",
                    "Replay transaction logs if available",
                    "Verify data integrity",
                    "Restart services and test"
                ]
            },
            
            "server_failure": {
                "severity": "critical",
                "estimated_downtime": "1-6 hours",
                "steps": [
                    "Activate backup server/infrastructure",
                    "Restore latest database backup",
                    "Restore application files",
                    "Update DNS/load balancer configuration",
                    "Test all critical functions",
                    "Monitor for issues"
                ]
            },
            
            "data_loss": {
                "severity": "high",
                "estimated_downtime": "1-3 hours",
                "steps": [
                    "Identify scope of data loss",
                    "Stop write operations",
                    "Restore from point-in-time backup",
                    "Recover any additional data from logs",
                    "Validate recovered data",
                    "Resume normal operations"
                ]
            }
        }
        
    def execute_recovery_procedure(self, disaster_type):
        """Execute disaster recovery procedure"""
        
        procedure = self.recovery_procedures.get(disaster_type)
        if not procedure:
            return {"error": f"Unknown disaster type: {disaster_type}"}
            
        recovery_log = {
            "disaster_type": disaster_type,
            "start_time": frappe.utils.now(),
            "severity": procedure["severity"],
            "estimated_downtime": procedure["estimated_downtime"],
            "steps_completed": [],
            "status": "in_progress"
        }
        
        try:
            for step_num, step in enumerate(procedure["steps"], 1):
                print(f"Step {step_num}: {step}")
                
                # Execute step-specific logic
                step_result = self.execute_recovery_step(disaster_type, step_num, step)
                
                recovery_log["steps_completed"].append({
                    "step_number": step_num,
                    "description": step,
                    "completed_at": frappe.utils.now(),
                    "result": step_result
                })
                
            recovery_log["status"] = "completed"
            recovery_log["end_time"] = frappe.utils.now()
            
        except Exception as e:
            recovery_log["status"] = "failed"
            recovery_log["error"] = str(e)
            recovery_log["failed_at"] = frappe.utils.now()
            
        return recovery_log
        
    def execute_recovery_step(self, disaster_type, step_num, step_description):
        """Execute individual recovery step"""
        
        # This would contain actual recovery logic
        # For now, return simulated results
        
        step_handlers = {
            ("database_corruption", 1): self.stop_services,
            ("database_corruption", 2): self.assess_db_corruption,
            ("database_corruption", 3): self.restore_database_backup,
            ("server_failure", 1): self.activate_backup_infrastructure,
            ("data_loss", 1): self.identify_data_loss_scope
        }
        
        handler = step_handlers.get((disaster_type, step_num))
        if handler:
            return handler()
        else:
            return {"status": "manual", "message": "Manual intervention required"}
            
    def stop_services(self):
        """Stop all application services"""
        try:
            # Stop supervisor services
            subprocess.run(["supervisorctl", "stop", "all"], check=True)
            
            # Stop nginx
            subprocess.run(["systemctl", "stop", "nginx"], check=True)
            
            return {"status": "success", "message": "All services stopped"}
        except Exception as e:
            return {"status": "error", "message": str(e)}
            
    def assess_db_corruption(self):
        """Assess database corruption extent"""
        corruption_report = {
            "corrupt_tables": [],
            "recoverable": True,
            "backup_required": True
        }
        
        try:
            # Check all tables
            tables = frappe.db.sql("SHOW TABLES", as_list=True)
            
            for table in tables:
                table_name = table[0]
                check_result = frappe.db.sql(f"CHECK TABLE `{table_name}`", as_dict=True)
                
                if check_result and check_result[0]["Msg_text"] != "OK":
                    corruption_report["corrupt_tables"].append({
                        "table": table_name,
                        "status": check_result[0]["Msg_text"]
                    })
                    
            return {
                "status": "success",
                "corruption_report": corruption_report
            }
            
        except Exception as e:
            return {"status": "error", "message": str(e)}
            
    def restore_database_backup(self):
        """Restore database from backup"""
        try:
            # Find latest backup
            backup_path = self.find_latest_backup()
            
            if not backup_path:
                return {"status": "error", "message": "No backup found"}
                
            # Restore database
            restore_command = f"mysql -u root -p{frappe.conf.db_root_password} {frappe.conf.db_name} < {backup_path}"
            result = subprocess.run(restore_command, shell=True, capture_output=True, text=True)
            
            if result.returncode == 0:
                return {
                    "status": "success",
                    "message": f"Database restored from {backup_path}",
                    "backup_file": backup_path
                }
            else:
                return {
                    "status": "error",
                    "message": f"Restore failed: {result.stderr}"
                }
                
        except Exception as e:
            return {"status": "error", "message": str(e)}
            
    def find_latest_backup(self):
        """Find latest database backup file"""
        
        backup_locations = [
            "/home/frappe/frappe-bench/sites/site1.local/private/backups",
            "/backup/database",
            "/var/backups/frappe"
        ]
        
        latest_backup = None
        latest_time = 0
        
        for location in backup_locations:
            if os.path.exists(location):
                for file in os.listdir(location):
                    if file.endswith('.sql.gz') or file.endswith('.sql'):
                        file_path = os.path.join(location, file)
                        file_time = os.path.getmtime(file_path)
                        
                        if file_time > latest_time:
                            latest_time = file_time
                            latest_backup = file_path
                            
        return latest_backup
        
    def activate_backup_infrastructure(self):
        """Activate backup server infrastructure"""
        # This would contain logic to:
        # - Spin up backup servers
        # - Update load balancer configuration
        # - Switch DNS records
        # For now, return a simulated result
        
        return {
            "status": "success",
            "message": "Backup infrastructure activated",
            "backup_servers": ["backup-1.example.com", "backup-2.example.com"]
        }
        
    def identify_data_loss_scope(self):
        """Identify scope of data loss"""
        
        try:
            # Check recent activity
            recent_activity = frappe.get_all("Activity Log",
                filters={"creation": [">=", frappe.utils.add_to_date(frappe.utils.now(), hours=-1)]},
                fields=["reference_doctype", "reference_name", "creation"],
                order_by="creation desc",
                limit=100
            )
            
            # Analyze affected doctypes
            affected_doctypes = {}
            for activity in recent_activity:
                doctype = activity["reference_doctype"]
                if doctype in affected_doctypes:
                    affected_doctypes[doctype] += 1
                else:
                    affected_doctypes[doctype] = 1
                    
            return {
                "status": "success",
                "recent_activity_count": len(recent_activity),
                "affected_doctypes": affected_doctypes,
                "time_range": "Last 1 hour"
            }
            
        except Exception as e:
            return {"status": "error", "message": str(e)}
            
    def create_recovery_report(self, recovery_log):
        """Create detailed recovery report"""
        
        report = {
            "summary": {
                "disaster_type": recovery_log["disaster_type"],
                "start_time": recovery_log["start_time"],
                "end_time": recovery_log.get("end_time"),
                "status": recovery_log["status"],
                "total_steps": len(recovery_log["steps_completed"])
            },
            "timeline": recovery_log["steps_completed"],
            "impact_assessment": self.assess_recovery_impact(recovery_log),
            "lessons_learned": self.generate_lessons_learned(recovery_log),
            "recommendations": self.generate_recovery_recommendations(recovery_log)
        }
        
        return report
        
    def assess_recovery_impact(self, recovery_log):
        """Assess impact of the disaster and recovery"""
        
        return {
            "downtime_duration": self.calculate_downtime(recovery_log),
            "data_loss_assessment": "To be determined through data validation",
            "service_availability": "Restored" if recovery_log["status"] == "completed" else "Partial",
            "user_impact": "All users affected during downtime"
        }
        
    def calculate_downtime(self, recovery_log):
        """Calculate total downtime duration"""
        
        if recovery_log.get("end_time"):
            start = frappe.utils.get_datetime(recovery_log["start_time"])
            end = frappe.utils.get_datetime(recovery_log["end_time"])
            duration = end - start
            return str(duration)
        else:
            return "Ongoing"
            
    def generate_lessons_learned(self, recovery_log):
        """Generate lessons learned from recovery"""
        
        lessons = [
            "Document all recovery procedures thoroughly",
            "Test disaster recovery procedures regularly",
            "Maintain up-to-date backup systems",
            "Implement monitoring for early issue detection"
        ]
        
        if recovery_log["status"] == "failed":
            lessons.extend([
                "Review and improve recovery procedures",
                "Identify gaps in recovery planning",
                "Consider additional backup strategies"
            ])
            
        return lessons
        
    def generate_recovery_recommendations(self, recovery_log):
        """Generate recommendations to prevent future disasters"""
        
        recommendations = [
            "Implement automated backup verification",
            "Set up comprehensive monitoring and alerting",
            "Create redundant infrastructure",
            "Conduct regular disaster recovery drills",
            "Maintain updated documentation",
            "Train staff on recovery procedures"
        ]
        
        return recommendations

# Recovery validation and testing
def validate_system_recovery():
    """Validate system after recovery"""
    
    validation_results = {
        "timestamp": frappe.utils.now(),
        "overall_status": "unknown",
        "checks": {}
    }
    
    # Database connectivity
    try:
        frappe.db.sql("SELECT 1")
        validation_results["checks"]["database"] = {
            "status": "pass",
            "message": "Database connectivity verified"
        }
    except Exception as e:
        validation_results["checks"]["database"] = {
            "status": "fail",
            "message": f"Database error: {str(e)}"
        }
        
    # Application functionality
    try:
        # Test basic operations
        user_count = frappe.db.count("User")
        doctype_count = frappe.db.count("DocType")
        
        validation_results["checks"]["application"] = {
            "status": "pass",
            "message": f"Application functional - {user_count} users, {doctype_count} doctypes"
        }
    except Exception as e:
        validation_results["checks"]["application"] = {
            "status": "fail",
            "message": f"Application error: {str(e)}"
        }
        
    # File system access
    try:
        test_file = frappe.get_site_path("recovery_test.txt")
        with open(test_file, 'w') as f:
            f.write("recovery test")
        os.remove(test_file)
        
        validation_results["checks"]["filesystem"] = {
            "status": "pass",
            "message": "File system access verified"
        }
    except Exception as e:
        validation_results["checks"]["filesystem"] = {
            "status": "fail",
            "message": f"File system error: {str(e)}"
        }
        
    # Determine overall status
    failed_checks = [
        check for check in validation_results["checks"].values()
        if check["status"] == "fail"
    ]
    
    if failed_checks:
        validation_results["overall_status"] = "fail"
    else:
        validation_results["overall_status"] = "pass"
        
    return validation_results
```

---

**Next Steps**: Continue with [Advanced Topics](12-advanced-topics.md) to explore framework internals, advanced patterns, performance optimization, and expert-level development techniques.