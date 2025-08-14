# ERPNext Module-Specific Patterns Analysis

## Table of Contents

1. [Overview](#overview)
2. [Accounts Module Patterns](#accounts-module-patterns)
3. [Stock Module Patterns](#stock-module-patterns)
4. [Manufacturing Module Patterns](#manufacturing-module-patterns)
5. [CRM Module Patterns](#crm-module-patterns)
6. [Selling Module Patterns](#selling-module-patterns)
7. [Buying Module Patterns](#buying-module-patterns)
8. [Projects Module Patterns](#projects-module-patterns)
9. [Assets Module Patterns](#assets-module-patterns)
10. [Setup Module Patterns](#setup-module-patterns)
11. [Cross-Module Integration Patterns](#cross-module-integration-patterns)
12. [Module Extension Patterns](#module-extension-patterns)

## Overview

ERPNext implements sophisticated module-specific patterns that handle the unique requirements of different business domains. Each module demonstrates specialized approaches to data modeling, business logic, workflow management, and integration patterns that have evolved from real-world enterprise needs.

### Core Module Design Principles

1. **Domain-Specific Logic**: Each module encapsulates business rules specific to its domain
2. **Integration Points**: Well-defined interfaces between modules for data flow
3. **Extensibility**: Patterns that allow customization without breaking core functionality
4. **Performance Optimization**: Module-specific optimizations for high-volume operations
5. **Workflow Orchestration**: Complex business process automation within and across modules

## Accounts Module Patterns

The Accounts module demonstrates sophisticated financial management patterns with multi-currency support, complex GL handling, and fiscal year management.

### Fiscal Year Management Pattern
*Reference: `/home/frappe/frappe-bench/apps/erpnext/erpnext/accounts/utils.py:58-81`*

```python
@frappe.whitelist()
def get_fiscal_year(
    date=None, fiscal_year=None, label="Date", verbose=1, company=None, as_dict=False, boolean=False
):
    """Comprehensive fiscal year management with caching and company-specific support"""
    if isinstance(boolean, str):
        boolean = loads(boolean)

    fiscal_years = get_fiscal_years(
        date, fiscal_year, label, verbose, company, as_dict=as_dict, boolean=boolean
    )
    if boolean:
        return fiscal_years
    else:
        return fiscal_years[0]

def get_fiscal_years(
    transaction_date=None,
    fiscal_year=None,
    label="Date",
    verbose=1,
    company=None,
    as_dict=False,
    boolean=False,
):
    """Advanced fiscal year retrieval with caching strategy"""
    fiscal_years = frappe.cache().hget("fiscal_years", company) or []

    if not fiscal_years:
        # Build dynamic query with company-specific fiscal years
        FY = DocType("Fiscal Year")
        query = (
            frappe.qb.from_(FY)
            .select(FY.name, FY.year_start_date, FY.year_end_date)
            .where(FY.disabled == 0)
        )

        if fiscal_year:
            query = query.where(FY.name == fiscal_year)

        if company:
            # Handle company-specific fiscal years
            FYC = DocType("Fiscal Year Company")
            query = query.where(
                ExistsCriterion(frappe.qb.from_(FYC).select(FYC.name).where(FYC.parent == FY.name)).negate()
                | ExistsCriterion(
                    frappe.qb.from_(FYC)
                    .select(FYC.name)
                    .where((FYC.parent == FY.name) & (FYC.company == company))
                )
            )

        fiscal_years = query.run(as_dict=True)
        frappe.cache().hset("fiscal_years", company, fiscal_years)

    return fiscal_years
```

### General Ledger Management Pattern

```python
class GLEntryManager:
    """Advanced GL entry management with batch processing and reconciliation"""
    
    def __init__(self, company=None):
        self.company = company
        self.gl_entries = []
        self.batch_size = 1000
    
    def create_gl_entries(self, doc, gl_map, cancel=False, adv_adj=False):
        """Create GL entries with comprehensive validation and processing"""
        
        # Validate GL map entries
        validated_entries = self.validate_gl_entries(gl_map)
        
        # Process in batches for performance
        for batch in self.create_batches(validated_entries):
            self.process_gl_batch(batch, doc, cancel, adv_adj)
    
    def validate_gl_entries(self, gl_map):
        """Validate GL entries before processing"""
        validated_entries = []
        
        for entry in gl_map:
            # Account validation
            if not frappe.db.exists("Account", entry.get("account")):
                frappe.throw(_("Account {0} does not exist").format(entry.get("account")))
            
            # Company validation
            account_company = frappe.db.get_value("Account", entry.get("account"), "company")
            if account_company != self.company:
                frappe.throw(_("Account {0} does not belong to company {1}").format(
                    entry.get("account"), self.company
                ))
            
            # Amount validation
            if not (entry.get("debit") or entry.get("credit")):
                frappe.throw(_("Either debit or credit amount must be specified"))
            
            validated_entries.append(entry)
        
        return validated_entries
    
    def process_gl_batch(self, batch, doc, cancel, adv_adj):
        """Process GL entries in batches with transaction management"""
        
        try:
            frappe.db.savepoint("gl_batch_processing")
            
            for entry in batch:
                gl_entry = self.create_gl_entry(entry, doc, cancel, adv_adj)
                gl_entry.insert(ignore_permissions=True)
                
        except Exception as e:
            frappe.db.rollback(save_point="gl_batch_processing")
            frappe.log_error("GL batch processing failed", str(e))
            raise
    
    def create_gl_entry(self, entry_data, doc, cancel, adv_adj):
        """Create individual GL entry with all required fields"""
        
        gl_entry = frappe.get_doc({
            "doctype": "GL Entry",
            "posting_date": entry_data.get("posting_date") or doc.posting_date,
            "transaction_date": entry_data.get("transaction_date") or doc.get("transaction_date"),
            "account": entry_data.get("account"),
            "party_type": entry_data.get("party_type"),
            "party": entry_data.get("party"),
            "cost_center": entry_data.get("cost_center"),
            "debit": flt(entry_data.get("debit")) if not cancel else flt(entry_data.get("credit")),
            "credit": flt(entry_data.get("credit")) if not cancel else flt(entry_data.get("debit")),
            "account_currency": entry_data.get("account_currency"),
            "debit_in_account_currency": entry_data.get("debit_in_account_currency"),
            "credit_in_account_currency": entry_data.get("credit_in_account_currency"),
            "against": entry_data.get("against"),
            "against_voucher_type": doc.doctype,
            "against_voucher": doc.name,
            "voucher_type": doc.doctype,
            "voucher_no": doc.name,
            "voucher_detail_no": entry_data.get("voucher_detail_no"),
            "project": entry_data.get("project") or doc.get("project"),
            "finance_book": entry_data.get("finance_book"),
            "company": self.company,
            "is_cancelled": 1 if cancel else 0
        })
        
        return gl_entry
```

### Multi-Currency Handling Pattern

```python
class CurrencyManager:
    """Advanced multi-currency management with real-time exchange rates"""
    
    def __init__(self, company):
        self.company = company
        self.company_currency = frappe.get_cached_value("Company", company, "default_currency")
    
    def get_exchange_rate(self, from_currency, to_currency, posting_date=None):
        """Get exchange rate with fallback mechanisms"""
        
        if from_currency == to_currency:
            return 1.0
        
        if not posting_date:
            posting_date = nowdate()
        
        # Try to get exchange rate from Currency Exchange
        exchange_rate = frappe.db.get_value(
            "Currency Exchange",
            {
                "from_currency": from_currency,
                "to_currency": to_currency,
                "date": posting_date
            },
            "exchange_rate"
        )
        
        if exchange_rate:
            return flt(exchange_rate)
        
        # Try reverse rate
        reverse_rate = frappe.db.get_value(
            "Currency Exchange",
            {
                "from_currency": to_currency,
                "to_currency": from_currency,
                "date": posting_date
            },
            "exchange_rate"
        )
        
        if reverse_rate:
            return 1.0 / flt(reverse_rate)
        
        # Get latest available rate
        latest_rate = self.get_latest_exchange_rate(from_currency, to_currency, posting_date)
        if latest_rate:
            return latest_rate
        
        # Fetch from external service if configured
        return self.fetch_external_exchange_rate(from_currency, to_currency, posting_date)
    
    def convert_currency(self, amount, from_currency, to_currency, posting_date=None, exchange_rate=None):
        """Convert amount between currencies with precision handling"""
        
        if from_currency == to_currency:
            return amount
        
        if not exchange_rate:
            exchange_rate = self.get_exchange_rate(from_currency, to_currency, posting_date)
        
        converted_amount = flt(amount) * flt(exchange_rate)
        
        # Apply currency precision
        currency_precision = get_currency_precision(to_currency)
        
        return flt(converted_amount, currency_precision)
    
    def get_currency_precision(self, currency):
        """Get precision for currency formatting"""
        return cint(frappe.db.get_value("Currency", currency, "fraction_units")) or 2
```

## Stock Module Patterns

The Stock module demonstrates sophisticated inventory management patterns with complex valuation strategies, serial/batch tracking, and warehouse hierarchies.

### Stock Valuation Pattern
*Reference: `/home/frappe/frappe-bench/apps/erpnext/erpnext/stock/utils.py:60-93`*

```python
def get_stock_value_on(
    warehouses: list | str | None = None,
    posting_date: str | None = None,
    item_code: str | None = None,
    company: str | None = None,
) -> float:
    """Advanced stock valuation with hierarchical warehouse support"""
    if not posting_date:
        posting_date = nowdate()

    sle = frappe.qb.DocType("Stock Ledger Entry")
    query = (
        frappe.qb.from_(sle)
        .select(IfNull(Sum(sle.stock_value_difference), 0))
        .where((sle.posting_date <= posting_date) & (sle.is_cancelled == 0))
    )

    if warehouses:
        if isinstance(warehouses, str):
            warehouses = [warehouses]

        # Expand warehouse groups to include all child warehouses
        warehouses = set(warehouses)
        for wh in list(warehouses):
            if frappe.db.get_value("Warehouse", wh, "is_group"):
                warehouses.update(get_child_warehouses(wh))

        query = query.where(sle.warehouse.isin(warehouses))

    if item_code:
        query = query.where(sle.item_code == item_code)

    if company:
        query = query.where(sle.company == company)

    return query.run(as_list=True)[0][0]
```

### Advanced Stock Valuation Strategies

```python
class StockValuationEngine:
    """Comprehensive stock valuation with multiple strategies"""
    
    def __init__(self, item_code, warehouse, company):
        self.item_code = item_code
        self.warehouse = warehouse
        self.company = company
        self.valuation_method = self.get_valuation_method()
    
    def get_valuation_method(self):
        """Get valuation method with fallback hierarchy"""
        
        # Item-specific valuation method
        item_valuation = frappe.db.get_value("Item", self.item_code, "valuation_method")
        if item_valuation:
            return item_valuation
        
        # Item group valuation method
        item_group = frappe.db.get_value("Item", self.item_code, "item_group")
        group_valuation = frappe.db.get_value("Item Group", item_group, "valuation_method")
        if group_valuation:
            return group_valuation
        
        # Company default valuation method
        company_valuation = frappe.db.get_value("Company", self.company, "default_valuation_method")
        return company_valuation or "FIFO"
    
    def calculate_stock_value(self, qty_to_value, posting_date, posting_time):
        """Calculate stock value based on valuation method"""
        
        if self.valuation_method == "FIFO":
            return self.calculate_fifo_value(qty_to_value, posting_date, posting_time)
        elif self.valuation_method == "LIFO":
            return self.calculate_lifo_value(qty_to_value, posting_date, posting_time)
        elif self.valuation_method == "Moving Average":
            return self.calculate_moving_average_value(qty_to_value, posting_date, posting_time)
        else:
            frappe.throw(_("Unknown valuation method: {0}").format(self.valuation_method))
    
    def calculate_fifo_value(self, qty_to_value, posting_date, posting_time):
        """FIFO valuation calculation with queue management"""
        
        # Get stock queue (oldest first)
        stock_queue = self.get_stock_queue(posting_date, posting_time)
        
        total_value = 0.0
        remaining_qty = qty_to_value
        
        for queue_item in stock_queue:
            if remaining_qty <= 0:
                break
            
            queue_qty = queue_item[0]
            queue_rate = queue_item[1]
            
            qty_to_consume = min(remaining_qty, queue_qty)
            total_value += qty_to_consume * queue_rate
            remaining_qty -= qty_to_consume
        
        return total_value
    
    def get_stock_queue(self, posting_date, posting_time):
        """Get stock queue for FIFO/LIFO calculations"""
        
        # Fetch all stock ledger entries up to the posting date/time
        sle_entries = frappe.db.sql("""
            SELECT qty_after_transaction, stock_value_difference, stock_queue
            FROM `tabStock Ledger Entry`
            WHERE item_code = %(item_code)s 
            AND warehouse = %(warehouse)s
            AND company = %(company)s
            AND (posting_date < %(posting_date)s 
                OR (posting_date = %(posting_date)s AND posting_time <= %(posting_time)s))
            AND is_cancelled = 0
            ORDER BY posting_date DESC, posting_time DESC, creation DESC
            LIMIT 1
        """, {
            "item_code": self.item_code,
            "warehouse": self.warehouse,
            "company": self.company,
            "posting_date": posting_date,
            "posting_time": posting_time
        }, as_dict=True)
        
        if sle_entries and sle_entries[0].stock_queue:
            import json
            return json.loads(sle_entries[0].stock_queue)
        
        return []

class SerialBatchManager:
    """Advanced serial and batch number management"""
    
    def __init__(self, item_code, warehouse):
        self.item_code = item_code
        self.warehouse = warehouse
        self.item_doc = frappe.get_cached_doc("Item", item_code)
    
    def process_serial_batch_bundle(self, bundle_data, voucher_type, voucher_no):
        """Process serial/batch bundle with validation"""
        
        if self.item_doc.has_serial_no:
            return self.process_serial_numbers(bundle_data, voucher_type, voucher_no)
        elif self.item_doc.has_batch_no:
            return self.process_batch_numbers(bundle_data, voucher_type, voucher_no)
    
    def process_serial_numbers(self, serial_nos, voucher_type, voucher_no):
        """Process serial numbers with status tracking"""
        
        processed_serials = []
        
        for serial_no in serial_nos:
            # Validate serial number exists
            if not frappe.db.exists("Serial No", serial_no):
                frappe.throw(_("Serial No {0} does not exist").format(serial_no))
            
            serial_doc = frappe.get_doc("Serial No", serial_no)
            
            # Validate serial number status
            if voucher_type in ["Sales Invoice", "Delivery Note"] and serial_doc.status != "Active":
                frappe.throw(_("Serial No {0} is not available for delivery").format(serial_no))
            
            # Update serial number status and location
            serial_doc.status = "Delivered" if voucher_type in ["Sales Invoice", "Delivery Note"] else "Active"
            serial_doc.warehouse = self.warehouse if voucher_type not in ["Sales Invoice", "Delivery Note"] else None
            serial_doc.save(ignore_permissions=True)
            
            processed_serials.append(serial_no)
        
        return processed_serials
    
    def process_batch_numbers(self, batch_data, voucher_type, voucher_no):
        """Process batch numbers with expiry and quantity tracking"""
        
        processed_batches = []
        
        for batch_item in batch_data:
            batch_no = batch_item.get("batch_no")
            qty = batch_item.get("qty")
            
            # Validate batch exists
            if not frappe.db.exists("Batch", batch_no):
                frappe.throw(_("Batch {0} does not exist").format(batch_no))
            
            batch_doc = frappe.get_doc("Batch", batch_no)
            
            # Validate batch expiry
            if batch_doc.expiry_date and getdate(batch_doc.expiry_date) < getdate():
                frappe.throw(_("Batch {0} has expired").format(batch_no))
            
            # Validate batch quantity availability
            available_qty = self.get_batch_qty(batch_no)
            if voucher_type in ["Sales Invoice", "Delivery Note"] and qty > available_qty:
                frappe.throw(_("Insufficient quantity in batch {0}").format(batch_no))
            
            processed_batches.append({
                "batch_no": batch_no,
                "qty": qty,
                "expiry_date": batch_doc.expiry_date
            })
        
        return processed_batches
```

## Manufacturing Module Patterns

The Manufacturing module demonstrates advanced production management patterns with complex BOM trees, work order orchestration, and capacity planning.

### BOM Tree Structure Pattern
*Reference: `/home/frappe/frappe-bench/apps/erpnext/erpnext/manufacturing/doctype/bom/bom.py:28-87`*

```python
class BOMTree:
    """Advanced BOM tree representation with recursive processing"""
    
    __slots__ = ["name", "child_items", "is_bom", "item_code", "qty", "exploded_qty", "bom_qty"]

    def __init__(self, name: str, is_bom: bool = True, exploded_qty: float = 1.0, qty: float = 1) -> None:
        self.name = name  # BOM number if is_bom else item_code
        self.child_items: list["BOMTree"] = []
        self.is_bom = is_bom
        self.item_code: str = None
        self.qty = qty  # required unit quantity
        self.exploded_qty = exploded_qty  # total exploded qty for root
        
        if not self.is_bom:
            self.item_code = self.name
        else:
            self.__create_tree()

    def __create_tree(self):
        """Recursively create BOM tree structure"""
        bom = frappe.get_cached_doc("BOM", self.name)
        self.item_code = bom.item
        self.bom_qty = bom.quantity

        for item in bom.get("items", []):
            qty = item.stock_qty / bom.quantity  # quantity per unit
            exploded_qty = self.exploded_qty * qty
            
            if item.bom_no:
                # Recursive BOM expansion
                child = BOMTree(item.bom_no, exploded_qty=exploded_qty, qty=qty)
                self.child_items.append(child)
            else:
                # Leaf item
                self.child_items.append(
                    BOMTree(item.item_code, is_bom=False, exploded_qty=exploded_qty, qty=qty)
                )

    def level_order_traversal(self) -> list["BOMTree"]:
        """Level-order traversal for processing sequence optimization"""
        traversal = []
        queue = deque()
        queue.append(self)

        while queue:
            node = queue.popleft()
            
            for child in node.child_items:
                traversal.append(child)
                queue.append(child)

        return traversal
```

### Work Order Management Pattern

```python
class WorkOrderManager:
    """Comprehensive work order management with capacity planning"""
    
    def __init__(self, work_order):
        self.work_order = work_order
        self.item_code = work_order.production_item
        self.bom = frappe.get_cached_doc("BOM", work_order.bom_no)
    
    def plan_production_schedule(self):
        """Advanced production scheduling with capacity constraints"""
        
        # Get workstation capacity
        workstation_capacity = self.get_workstation_capacity()
        
        # Calculate operation sequences
        operations = self.plan_operations_sequence()
        
        # Schedule with capacity constraints
        scheduled_operations = self.apply_capacity_constraints(operations, workstation_capacity)
        
        # Create job cards
        self.create_job_cards(scheduled_operations)
        
        return scheduled_operations
    
    def get_workstation_capacity(self):
        """Get workstation capacity with holiday considerations"""
        
        capacity_map = {}
        
        for operation in self.bom.operations:
            workstation = operation.workstation
            
            if workstation not in capacity_map:
                capacity_data = frappe.get_cached_doc("Workstation", workstation)
                
                # Calculate effective capacity
                working_hours = self.calculate_working_hours(capacity_data)
                efficiency_factor = capacity_data.efficiency / 100.0
                
                capacity_map[workstation] = {
                    "daily_capacity": working_hours * efficiency_factor,
                    "hourly_capacity": efficiency_factor,
                    "holiday_list": capacity_data.holiday_list
                }
        
        return capacity_map
    
    def plan_operations_sequence(self):
        """Plan operation sequence with dependency management"""
        
        operations = []
        
        for operation in self.bom.operations:
            operation_time = self.calculate_operation_time(operation)
            
            operations.append({
                "operation": operation.operation,
                "workstation": operation.workstation,
                "time_in_mins": operation_time,
                "sequence_id": operation.sequence_id,
                "dependencies": self.get_operation_dependencies(operation)
            })
        
        # Sort by sequence and dependencies
        return self.sort_operations_by_sequence(operations)
    
    def apply_capacity_constraints(self, operations, capacity_map):
        """Apply capacity constraints to operation scheduling"""
        
        scheduled_operations = []
        workstation_schedules = defaultdict(list)
        
        for operation in operations:
            workstation = operation["workstation"]
            required_time = operation["time_in_mins"] / 60.0  # Convert to hours
            
            # Find available slot
            available_slot = self.find_available_slot(
                workstation, required_time, capacity_map[workstation], workstation_schedules
            )
            
            operation.update({
                "scheduled_start": available_slot["start_time"],
                "scheduled_end": available_slot["end_time"],
                "planned_start_time": available_slot["start_time"],
                "planned_end_time": available_slot["end_time"]
            })
            
            # Update workstation schedule
            workstation_schedules[workstation].append(available_slot)
            scheduled_operations.append(operation)
        
        return scheduled_operations
    
    def create_job_cards(self, scheduled_operations):
        """Create job cards for scheduled operations"""
        
        job_cards = []
        
        for operation in scheduled_operations:
            job_card = frappe.get_doc({
                "doctype": "Job Card",
                "work_order": self.work_order.name,
                "operation": operation["operation"],
                "workstation": operation["workstation"],
                "planned_start_time": operation["planned_start_time"],
                "planned_end_time": operation["planned_end_time"],
                "for_quantity": self.work_order.qty,
                "operation_sequence": operation["sequence_id"],
                "time_required": operation["time_in_mins"]
            })
            
            job_card.insert()
            job_cards.append(job_card)
        
        return job_cards

class ProductionPlanningEngine:
    """Advanced production planning with MRP logic"""
    
    def __init__(self, company):
        self.company = company
    
    def create_production_plan(self, sales_orders, material_requests=None):
        """Create comprehensive production plan with MRP calculations"""
        
        # Aggregate demand from sales orders and material requests
        demand_map = self.aggregate_demand(sales_orders, material_requests)
        
        # Calculate net requirements with inventory considerations
        net_requirements = self.calculate_net_requirements(demand_map)
        
        # Generate production schedule
        production_schedule = self.generate_production_schedule(net_requirements)
        
        # Create production plan document
        production_plan = self.create_production_plan_document(production_schedule)
        
        return production_plan
    
    def calculate_net_requirements(self, demand_map):
        """Calculate net requirements considering inventory and planned production"""
        
        net_requirements = {}
        
        for item_code, demand_qty in demand_map.items():
            # Get current stock
            current_stock = get_stock_balance(item_code, warehouse=None, company=self.company)
            
            # Get planned receipts from existing work orders
            planned_receipts = self.get_planned_receipts(item_code)
            
            # Calculate net requirement
            net_requirement = demand_qty - current_stock - planned_receipts
            
            if net_requirement > 0:
                net_requirements[item_code] = net_requirement
        
        return net_requirements
```

## CRM Module Patterns

The CRM module demonstrates sophisticated customer relationship management patterns with lead qualification, opportunity tracking, and campaign automation.

### Lead Conversion Pattern
*Reference: `/home/frappe/frappe-bench/apps/erpnext/erpnext/crm/doctype/lead/lead.py:20-98`*

```python
class Lead(SellingController, CRMNote):
    """Advanced lead management with conversion tracking and automation"""
    
    def validate(self):
        """Comprehensive lead validation with business rules"""
        self.set_full_name()
        self.set_lead_name()
        self.set_title()
        self.set_status()
        self.check_email_id_is_unique()
        self.validate_email_id()
    
    def set_status(self):
        """Intelligent status management based on activities"""
        if self.status != "Lead":
            return
        
        # Check for associated opportunities
        opportunities = frappe.get_all("Opportunity", 
                                     filters={"lead": self.name, "status": "Open"})
        if opportunities:
            self.status = "Opportunity"
            return
        
        # Check for quotations
        quotations = frappe.get_all("Quotation", 
                                  filters={"lead": self.name, "docstatus": ["<", 2]})
        if quotations:
            self.status = "Quotation"
            return
        
        # Check communication activity
        recent_communications = frappe.get_all("Communication",
            filters={
                "reference_doctype": self.doctype,
                "reference_name": self.name,
                "creation": [">=", add_days(today(), -7)]
            }
        )
        
        if recent_communications:
            self.status = "Replied"
```

### Advanced CRM Automation Engine

```python
class CRMAutomationEngine:
    """Comprehensive CRM automation with lead scoring and nurturing"""
    
    def __init__(self, lead_doc):
        self.lead = lead_doc
        self.scoring_rules = self.get_lead_scoring_rules()
    
    def calculate_lead_score(self):
        """Calculate lead score based on multiple factors"""
        
        total_score = 0
        score_breakdown = {}
        
        # Demographic scoring
        demo_score = self.calculate_demographic_score()
        total_score += demo_score
        score_breakdown["demographic"] = demo_score
        
        # Behavioral scoring
        behavior_score = self.calculate_behavioral_score()
        total_score += behavior_score
        score_breakdown["behavioral"] = behavior_score
        
        # Company scoring
        company_score = self.calculate_company_score()
        total_score += company_score
        score_breakdown["company"] = company_score
        
        # Engagement scoring
        engagement_score = self.calculate_engagement_score()
        total_score += engagement_score
        score_breakdown["engagement"] = engagement_score
        
        return {
            "total_score": total_score,
            "breakdown": score_breakdown,
            "qualification_level": self.get_qualification_level(total_score)
        }
    
    def calculate_demographic_score(self):
        """Score based on demographic information"""
        score = 0
        
        # Job title scoring
        if self.lead.job_title:
            job_title_scores = {
                "CEO": 25, "President": 25, "VP": 20, "Director": 15,
                "Manager": 10, "Analyst": 5
            }
            
            for title, title_score in job_title_scores.items():
                if title.lower() in self.lead.job_title.lower():
                    score += title_score
                    break
        
        # Industry scoring
        if self.lead.industry:
            industry_scores = self.get_industry_scores()
            score += industry_scores.get(self.lead.industry, 0)
        
        # Company size scoring
        if self.lead.no_of_employees:
            size_scores = {
                "1000+": 25, "501-1000": 20, "201-500": 15,
                "51-200": 10, "11-50": 5, "1-10": 2
            }
            score += size_scores.get(self.lead.no_of_employees, 0)
        
        return score
    
    def calculate_behavioral_score(self):
        """Score based on behavioral indicators"""
        score = 0
        
        # Website activity
        website_visits = self.get_website_activity()
        score += min(website_visits * 2, 20)  # Cap at 20 points
        
        # Email engagement
        email_opens = self.get_email_opens()
        email_clicks = self.get_email_clicks()
        score += min(email_opens * 1 + email_clicks * 3, 25)  # Cap at 25 points
        
        # Content engagement
        content_downloads = self.get_content_downloads()
        score += min(content_downloads * 5, 20)  # Cap at 20 points
        
        return score
    
    def trigger_automation_actions(self, lead_score):
        """Trigger automated actions based on lead score"""
        
        automation_actions = []
        
        if lead_score["total_score"] >= 80:
            # High-value lead actions
            automation_actions.extend([
                {"action": "assign_to_sales_rep", "priority": "high"},
                {"action": "schedule_call", "delay": 0},
                {"action": "send_personalized_proposal", "delay": 24}
            ])
        elif lead_score["total_score"] >= 60:
            # Medium-value lead actions
            automation_actions.extend([
                {"action": "assign_to_sales_rep", "priority": "medium"},
                {"action": "send_targeted_content", "delay": 2},
                {"action": "schedule_demo", "delay": 72}
            ])
        elif lead_score["total_score"] >= 40:
            # Nurturing actions
            automation_actions.extend([
                {"action": "add_to_nurture_campaign", "campaign_type": "educational"},
                {"action": "send_case_study", "delay": 24}
            ])
        else:
            # Low engagement - minimal actions
            automation_actions.append({
                "action": "add_to_nurture_campaign", "campaign_type": "awareness"
            })
        
        # Execute automation actions
        for action in automation_actions:
            self.execute_automation_action(action)
        
        return automation_actions

class OpportunityManagementEngine:
    """Advanced opportunity management with predictive analytics"""
    
    def __init__(self, opportunity_doc):
        self.opportunity = opportunity_doc
        self.historical_data = self.get_historical_data()
    
    def calculate_probability(self):
        """Calculate win probability using historical data and ML"""
        
        # Factors affecting probability
        factors = {
            "sales_stage": self.get_stage_probability(),
            "deal_size": self.get_size_probability(),
            "lead_source": self.get_source_probability(),
            "sales_rep": self.get_rep_probability(),
            "timeline": self.get_timeline_probability(),
            "competition": self.get_competition_probability()
        }
        
        # Weighted probability calculation
        weights = {
            "sales_stage": 0.3,
            "deal_size": 0.2,
            "lead_source": 0.15,
            "sales_rep": 0.15,
            "timeline": 0.1,
            "competition": 0.1
        }
        
        probability = sum(factors[factor] * weights[factor] for factor in factors)
        
        return {
            "probability": probability,
            "factors": factors,
            "confidence_level": self.calculate_confidence_level(factors)
        }
    
    def recommend_next_actions(self):
        """Recommend next actions based on opportunity state"""
        
        recommendations = []
        
        # Stage-specific recommendations
        if self.opportunity.sales_stage == "Qualification":
            recommendations.extend([
                {"action": "needs_analysis", "priority": "high"},
                {"action": "budget_confirmation", "priority": "high"},
                {"action": "decision_maker_identification", "priority": "medium"}
            ])
        elif self.opportunity.sales_stage == "Proposal":
            recommendations.extend([
                {"action": "proposal_presentation", "priority": "high"},
                {"action": "technical_evaluation", "priority": "medium"},
                {"action": "roi_analysis", "priority": "medium"}
            ])
        elif self.opportunity.sales_stage == "Negotiation":
            recommendations.extend([
                {"action": "contract_terms_discussion", "priority": "high"},
                {"action": "pricing_negotiation", "priority": "high"},
                {"action": "legal_review", "priority": "medium"}
            ])
        
        # Time-sensitive recommendations
        days_since_update = (getdate() - getdate(self.opportunity.modified)).days
        if days_since_update > 7:
            recommendations.append({
                "action": "follow_up_call",
                "priority": "high",
                "reason": "No activity for over 7 days"
            })
        
        return recommendations
```

## Cross-Module Integration Patterns

### Sales-to-Manufacturing Integration

```python
class SalesManufacturingIntegrator:
    """Integrate sales orders with manufacturing planning"""
    
    def process_sales_order_for_manufacturing(self, sales_order):
        """Process sales order items for manufacturing requirements"""
        
        manufacturing_items = []
        purchase_items = []
        
        for item in sales_order.items:
            item_doc = frappe.get_cached_doc("Item", item.item_code)
            
            if item_doc.is_stock_item and item_doc.default_bom:
                # Check if we need to manufacture
                available_stock = get_stock_balance(item.item_code)
                required_qty = item.qty - available_stock
                
                if required_qty > 0:
                    manufacturing_items.append({
                        "item_code": item.item_code,
                        "bom_no": item_doc.default_bom,
                        "qty_to_manufacture": required_qty,
                        "delivery_date": item.delivery_date,
                        "sales_order": sales_order.name,
                        "sales_order_item": item.name
                    })
            elif item_doc.is_purchase_item:
                purchase_items.append(item)
        
        # Create manufacturing entries
        if manufacturing_items:
            self.create_production_plan_items(manufacturing_items, sales_order)
        
        # Create purchase requests
        if purchase_items:
            self.create_material_request(purchase_items, sales_order)
    
    def create_production_plan_items(self, manufacturing_items, sales_order):
        """Create production plan items from sales order requirements"""
        
        # Check for existing production plan
        existing_plan = frappe.db.get_value(
            "Production Plan", 
            {"sales_order": sales_order.name, "docstatus": 0}
        )
        
        if existing_plan:
            production_plan = frappe.get_doc("Production Plan", existing_plan)
        else:
            production_plan = frappe.get_doc({
                "doctype": "Production Plan",
                "company": sales_order.company,
                "sales_order": sales_order.name,
                "posting_date": today()
            })
        
        # Add manufacturing items to production plan
        for item in manufacturing_items:
            production_plan.append("po_items", {
                "item_code": item["item_code"],
                "bom_no": item["bom_no"],
                "planned_qty": item["qty_to_manufacture"],
                "planned_start_date": add_days(today(), -3),  # Lead time consideration
                "sales_order": item["sales_order"],
                "sales_order_item": item["sales_order_item"]
            })
        
        if not existing_plan:
            production_plan.insert()
        else:
            production_plan.save()
        
        return production_plan

class FinancialIntegrationEngine:
    """Advanced financial integration across modules"""
    
    def __init__(self, company):
        self.company = company
        self.gl_manager = GLEntryManager(company)
    
    def process_stock_to_finance_integration(self, stock_entry):
        """Process stock entries for financial impact"""
        
        gl_entries = []
        
        for item in stock_entry.items:
            # Stock valuation entries
            if stock_entry.purpose == "Material Receipt":
                gl_entries.extend(self.create_material_receipt_gl_entries(item, stock_entry))
            elif stock_entry.purpose == "Material Issue":
                gl_entries.extend(self.create_material_issue_gl_entries(item, stock_entry))
            elif stock_entry.purpose == "Material Transfer":
                gl_entries.extend(self.create_material_transfer_gl_entries(item, stock_entry))
        
        # Create GL entries
        if gl_entries:
            self.gl_manager.create_gl_entries(stock_entry, gl_entries)
    
    def create_material_receipt_gl_entries(self, item, stock_entry):
        """Create GL entries for material receipt"""
        
        warehouse_account = get_warehouse_account_map().get(item.t_warehouse)
        
        return [
            {
                "account": warehouse_account["account"],
                "debit": item.amount,
                "cost_center": item.cost_center,
                "against": "Temporary Opening"
            },
            {
                "account": "Temporary Opening",
                "credit": item.amount,
                "cost_center": item.cost_center,
                "against": warehouse_account["account"]
            }
        ]
```

## Selling Module Patterns

The Selling module demonstrates advanced sales process management patterns with document mapping, pricing strategies, and territory-based sales management.

### Document Mapping Pattern
*Reference: `/home/frappe/frappe-bench/apps/erpnext/erpnext/crm/doctype/lead/lead.py:312-350`*

```python
@frappe.whitelist()
def make_customer(source_name, target_doc=None):
    """Intelligent document mapping from Lead to Customer"""
    return _make_customer(source_name, target_doc)

def _make_customer(source_name, target_doc=None, ignore_permissions=False):
    def set_missing_values(source, target):
        """Set contextual values based on lead type"""
        if source.company_name:
            target.customer_type = "Company"
            target.customer_name = source.company_name
        else:
            target.customer_type = "Individual"
            target.customer_name = source.lead_name

        if not target.customer_group:
            target.customer_group = frappe.db.get_default("Customer Group")

    doclist = get_mapped_doc(
        "Lead",
        source_name,
        {
            "Lead": {
                "doctype": "Customer",
                "field_map": {
                    "name": "lead_name",
                    "company_name": "customer_name",
                    "contact_no": "phone_1",
                    "fax": "fax_1",
                },
                "field_no_map": ["disabled"],
            }
        },
        target_doc,
        set_missing_values,
        ignore_permissions=ignore_permissions,
    )

    return doclist
```

### Advanced Pricing Engine

```python
class AdvancedPricingEngine:
    """Comprehensive pricing engine with dynamic rule evaluation"""
    
    def __init__(self, item_code, customer=None, transaction_date=None):
        self.item_code = item_code
        self.customer = customer
        self.transaction_date = transaction_date or today()
        self.pricing_rules = self.get_applicable_pricing_rules()
    
    def calculate_price(self, qty=1, uom=None):
        """Calculate price with multiple pricing strategies"""
        
        base_price = self.get_base_price(uom)
        
        # Apply quantity-based pricing
        qty_price = self.apply_quantity_breaks(base_price, qty)
        
        # Apply customer-specific pricing
        customer_price = self.apply_customer_pricing(qty_price)
        
        # Apply promotional pricing
        promotional_price = self.apply_promotional_pricing(customer_price)
        
        # Apply currency conversion if needed
        final_price = self.apply_currency_conversion(promotional_price)
        
        return {
            "rate": final_price,
            "base_rate": base_price,
            "pricing_breakdown": {
                "quantity_adjustment": qty_price - base_price,
                "customer_adjustment": customer_price - qty_price,
                "promotional_adjustment": promotional_price - customer_price,
                "currency_adjustment": final_price - promotional_price
            }
        }
    
    def apply_quantity_breaks(self, base_price, qty):
        """Apply quantity-based pricing breaks"""
        
        # Get quantity breaks for item
        qty_breaks = frappe.get_all(
            "Item Price",
            filters={
                "item_code": self.item_code,
                "min_qty": ["<=", qty],
                "valid_from": ["<=", self.transaction_date],
                "valid_upto": [">=", self.transaction_date]
            },
            order_by="min_qty desc",
            limit=1
        )
        
        if qty_breaks:
            qty_price = frappe.db.get_value("Item Price", qty_breaks[0].name, "price_list_rate")
            return flt(qty_price)
        
        return base_price
    
    def apply_customer_pricing(self, base_price):
        """Apply customer-specific pricing rules"""
        
        if not self.customer:
            return base_price
        
        # Check for customer-specific item prices
        customer_price = frappe.db.get_value(
            "Item Price",
            {
                "item_code": self.item_code,
                "customer": self.customer,
                "valid_from": ["<=", self.transaction_date],
                "valid_upto": [">=", self.transaction_date]
            },
            "price_list_rate"
        )
        
        if customer_price:
            return flt(customer_price)
        
        # Check customer group pricing
        customer_group = frappe.db.get_value("Customer", self.customer, "customer_group")
        if customer_group:
            group_price = frappe.db.get_value(
                "Item Price",
                {
                    "item_code": self.item_code,
                    "customer_group": customer_group,
                    "valid_from": ["<=", self.transaction_date],
                    "valid_upto": [">=", self.transaction_date]
                },
                "price_list_rate"
            )
            
            if group_price:
                return flt(group_price)
        
        return base_price

class TerritoryManager:
    """Advanced territory-based sales management"""
    
    def __init__(self, company):
        self.company = company
        self.territory_tree = self.build_territory_tree()
    
    def assign_sales_person_by_territory(self, customer, territory=None):
        """Assign sales person based on territory hierarchy"""
        
        if not territory:
            territory = frappe.db.get_value("Customer", customer, "territory")
        
        if not territory:
            return None
        
        # Find sales person assigned to territory or parent territory
        sales_person = self.find_territory_sales_person(territory)
        
        return sales_person
    
    def find_territory_sales_person(self, territory):
        """Find sales person assigned to territory using hierarchy"""
        
        # Check direct assignment
        sales_person = frappe.db.get_value(
            "Sales Person Territory",
            {"territory": territory},
            "parent"
        )
        
        if sales_person:
            return sales_person
        
        # Check parent territory
        parent_territory = frappe.db.get_value("Territory", territory, "parent_territory")
        
        if parent_territory and parent_territory != "All Territories":
            return self.find_territory_sales_person(parent_territory)
        
        return None
```

## Buying Module Patterns

### Purchase Request Management

```python
class PurchaseRequestManager:
    """Advanced purchase request management with approval workflows"""
    
    def __init__(self, company):
        self.company = company
        self.approval_matrix = self.get_approval_matrix()
    
    def create_purchase_requests_from_reorder(self):
        """Create purchase requests based on reorder levels"""
        
        # Get items below reorder level
        low_stock_items = self.get_items_below_reorder_level()
        
        # Group by supplier for consolidated requests
        supplier_items = self.group_items_by_supplier(low_stock_items)
        
        # Create purchase requests
        purchase_requests = []
        for supplier, items in supplier_items.items():
            pr = self.create_purchase_request(supplier, items)
            purchase_requests.append(pr)
        
        return purchase_requests
    
    def get_items_below_reorder_level(self):
        """Identify items that need reordering"""
        
        query = """
            SELECT 
                bin.item_code,
                bin.warehouse,
                bin.actual_qty,
                item.reorder_level,
                item.reorder_qty,
                item.default_supplier
            FROM `tabBin` bin
            JOIN `tabItem` item ON bin.item_code = item.name
            WHERE 
                bin.actual_qty <= item.reorder_level
                AND item.is_purchase_item = 1
                AND item.disabled = 0
                AND item.reorder_level > 0
                AND bin.warehouse IN (
                    SELECT name FROM `tabWarehouse` 
                    WHERE company = %(company)s
                )
        """
        
        return frappe.db.sql(query, {"company": self.company}, as_dict=True)
    
    def apply_approval_workflow(self, purchase_request):
        """Apply dynamic approval workflow based on amount"""
        
        total_amount = sum(item.amount for item in purchase_request.items)
        
        # Determine approval level based on amount
        approval_level = self.get_approval_level(total_amount)
        
        # Create approval workflow
        workflow_doc = frappe.get_doc({
            "doctype": "Workflow Action",
            "reference_doctype": "Purchase Request",
            "reference_name": purchase_request.name,
            "workflow_state": "Pending Approval",
            "next_action_user": approval_level["approver"],
            "action_taken": "Submit for Approval",
            "comments": f"Purchase request for {total_amount} requires {approval_level['level']} approval"
        })
        
        workflow_doc.insert()
        
        # Send notification to approver
        self.send_approval_notification(purchase_request, approval_level)
    
    def get_approval_level(self, amount):
        """Get required approval level based on amount"""
        
        approval_levels = [
            {"threshold": 100000, "level": "CEO", "approver": self.get_ceo_user()},
            {"threshold": 50000, "level": "Finance Manager", "approver": self.get_finance_manager()},
            {"threshold": 10000, "level": "Purchase Manager", "approver": self.get_purchase_manager()},
            {"threshold": 0, "level": "Department Head", "approver": self.get_department_head()}
        ]
        
        for level in approval_levels:
            if amount >= level["threshold"]:
                return level
        
        return approval_levels[-1]  # Default to lowest level
```

## Stock Module Advanced Patterns

### Advanced Stock Utility Functions
*Reference: `/home/frappe/frappe-bench/apps/erpnext/erpnext/stock/utils.py:96-161`*

```python
@frappe.whitelist()
def get_stock_balance(
    item_code,
    warehouse,
    posting_date=None,
    posting_time=None,
    with_valuation_rate=False,
    with_serial_no=False,
    inventory_dimensions_dict=None,
):
    """Comprehensive stock balance calculation with multiple options"""
    
    from erpnext.stock.stock_ledger import get_previous_sle

    frappe.has_permission("Item", "read", throw=True)

    if posting_date is None:
        posting_date = nowdate()
    if posting_time is None:
        posting_time = nowtime()

    args = {
        "item_code": item_code,
        "warehouse": warehouse,
        "posting_date": posting_date,
        "posting_time": posting_time,
    }

    # Handle inventory dimensions for multi-dimensional stock tracking
    extra_cond = ""
    if inventory_dimensions_dict:
        for field, value in inventory_dimensions_dict.items():
            args[field] = value
            extra_cond += f" and {field} = %({field})s"

    last_entry = get_previous_sle(args, extra_cond=extra_cond)

    if with_valuation_rate:
        if with_serial_no:
            # Advanced serial number tracking with availability
            serial_no_details = get_available_serial_nos(
                frappe._dict({
                    "item_code": item_code,
                    "warehouse": warehouse,
                    "posting_date": posting_date,
                    "posting_time": posting_time,
                    "ignore_warehouse": 1,
                })
            )

            serial_nos = ""
            if serial_no_details:
                serial_nos = "\n".join(d.serial_no for d in serial_no_details)

            return (
                (last_entry.qty_after_transaction, last_entry.valuation_rate, serial_nos)
                if last_entry
                else (0.0, 0.0, None)
            )
        else:
            return (last_entry.qty_after_transaction, last_entry.valuation_rate) if last_entry else (0.0, 0.0)
    else:
        return last_entry.qty_after_transaction if last_entry else 0.0
```

### Barcode Scanning System
*Reference: `/home/frappe/frappe-bench/apps/erpnext/erpnext/stock/utils.py:586-642`*

```python
@frappe.whitelist()
def scan_barcode(search_value: str) -> BarcodeScanResult:
    """Intelligent barcode scanning with caching and multiple lookup strategies"""
    
    def set_cache(data: BarcodeScanResult):
        frappe.cache().set_value(f"erpnext:barcode_scan:{search_value}", data, expires_in_sec=120)

    def get_cache() -> BarcodeScanResult | None:
        if data := frappe.cache().get_value(f"erpnext:barcode_scan:{search_value}"):
            return data

    # Check cache first for performance
    if scan_data := get_cache():
        return scan_data

    # Multi-strategy barcode lookup
    
    # Strategy 1: Search barcode table
    barcode_data = frappe.db.get_value(
        "Item Barcode",
        {"barcode": search_value},
        ["barcode", "parent as item_code", "uom"],
        as_dict=True,
    )
    if barcode_data:
        _update_item_info(barcode_data)
        set_cache(barcode_data)
        return barcode_data

    # Strategy 2: Search serial numbers
    serial_no_data = frappe.db.get_value(
        "Serial No",
        search_value,
        ["name as serial_no", "item_code", "batch_no"],
        as_dict=True,
    )
    if serial_no_data:
        _update_item_info(serial_no_data)
        set_cache(serial_no_data)
        return serial_no_data

    # Strategy 3: Search batch numbers
    batch_no_data = frappe.db.get_value(
        "Batch",
        search_value,
        ["name as batch_no", "item as item_code"],
        as_dict=True,
    )
    if batch_no_data:
        # Validate batch-serial compatibility
        if frappe.get_cached_value("Item", batch_no_data.item_code, "has_serial_no"):
            frappe.throw(
                _(
                    "Batch No {0} is linked with Item {1} which has serial no. Please scan serial no instead."
                ).format(search_value, batch_no_data.item_code)
            )

        _update_item_info(batch_no_data)
        set_cache(batch_no_data)
        return batch_no_data

    return {}

def _update_item_info(scan_result: dict[str, str | None]) -> dict[str, str | None]:
    """Enrich scan result with item information"""
    if item_code := scan_result.get("item_code"):
        if item_info := frappe.get_cached_value(
            "Item",
            item_code,
            ["has_batch_no", "has_serial_no"],
            as_dict=True,
        ):
            scan_result.update(item_info)
    return scan_result
```

## Projects Module Patterns

### Project Management Engine

```python
class ProjectManagementEngine:
    """Comprehensive project management with resource allocation"""
    
    def __init__(self, project_name):
        self.project = frappe.get_doc("Project", project_name)
        self.resource_pool = self.get_available_resources()
    
    def create_project_plan(self, tasks_data):
        """Create detailed project plan with dependencies"""
        
        # Create task hierarchy
        task_tree = self.build_task_tree(tasks_data)
        
        # Calculate critical path
        critical_path = self.calculate_critical_path(task_tree)
        
        # Allocate resources
        resource_allocation = self.allocate_resources(task_tree)
        
        # Create project timeline
        timeline = self.create_project_timeline(task_tree, critical_path)
        
        return {
            "task_tree": task_tree,
            "critical_path": critical_path,
            "resource_allocation": resource_allocation,
            "timeline": timeline
        }
    
    def track_project_progress(self):
        """Track project progress with earned value analysis"""
        
        tasks = frappe.get_all(
            "Task",
            filters={"project": self.project.name},
            fields=["name", "progress", "expected_time", "actual_time", "status"]
        )
        
        # Calculate project metrics
        total_planned_hours = sum(task.expected_time or 0 for task in tasks)
        total_actual_hours = sum(task.actual_time or 0 for task in tasks)
        completed_work = sum(
            (task.expected_time or 0) * (task.progress or 0) / 100 
            for task in tasks
        )
        
        # Earned Value Analysis
        planned_value = total_planned_hours * (self.project.percent_complete or 0) / 100
        earned_value = completed_work
        actual_cost = total_actual_hours
        
        return {
            "planned_value": planned_value,
            "earned_value": earned_value,
            "actual_cost": actual_cost,
            "schedule_variance": earned_value - planned_value,
            "cost_variance": earned_value - actual_cost,
            "schedule_performance_index": earned_value / planned_value if planned_value else 0,
            "cost_performance_index": earned_value / actual_cost if actual_cost else 0
        }
```

## Assets Module Patterns

### Asset Management System

```python
class AssetManagementSystem:
    """Comprehensive asset lifecycle management"""
    
    def __init__(self, company):
        self.company = company
    
    def calculate_depreciation_schedule(self, asset):
        """Calculate comprehensive depreciation schedule"""
        
        depreciation_method = asset.depreciation_method
        useful_life = asset.total_number_of_depreciations
        asset_value = asset.gross_purchase_amount
        salvage_value = asset.expected_value_after_useful_life or 0
        
        schedule = []
        
        if depreciation_method == "Straight Line":
            annual_depreciation = (asset_value - salvage_value) / useful_life
            
            for year in range(1, useful_life + 1):
                schedule.append({
                    "period": year,
                    "depreciation_amount": annual_depreciation,
                    "accumulated_depreciation": annual_depreciation * year,
                    "asset_value": asset_value - (annual_depreciation * year)
                })
        
        elif depreciation_method == "Double Declining Balance":
            depreciation_rate = 2 / useful_life
            remaining_value = asset_value
            accumulated_depreciation = 0
            
            for year in range(1, useful_life + 1):
                # Ensure we don't depreciate below salvage value
                max_depreciation = remaining_value - salvage_value
                annual_depreciation = min(
                    remaining_value * depreciation_rate,
                    max_depreciation
                )
                
                accumulated_depreciation += annual_depreciation
                remaining_value -= annual_depreciation
                
                schedule.append({
                    "period": year,
                    "depreciation_amount": annual_depreciation,
                    "accumulated_depreciation": accumulated_depreciation,
                    "asset_value": remaining_value
                })
                
                if remaining_value <= salvage_value:
                    break
        
        return schedule
    
    def track_asset_maintenance(self, asset_name):
        """Track asset maintenance schedules and costs"""
        
        asset = frappe.get_doc("Asset", asset_name)
        
        # Get maintenance schedules
        maintenance_schedules = frappe.get_all(
            "Asset Maintenance",
            filters={"asset_name": asset_name},
            fields=["maintenance_type", "periodicity", "next_due_date"]
        )
        
        # Calculate maintenance costs
        maintenance_logs = frappe.get_all(
            "Asset Maintenance Log",
            filters={"asset_name": asset_name},
            fields=["maintenance_date", "maintenance_cost", "maintenance_type"]
        )
        
        total_maintenance_cost = sum(log.maintenance_cost or 0 for log in maintenance_logs)
        
        # Predict next maintenance
        upcoming_maintenance = []
        for schedule in maintenance_schedules:
            if schedule.next_due_date and getdate(schedule.next_due_date) <= add_days(today(), 30):
                upcoming_maintenance.append(schedule)
        
        return {
            "total_maintenance_cost": total_maintenance_cost,
            "maintenance_history": maintenance_logs,
            "upcoming_maintenance": upcoming_maintenance,
            "maintenance_cost_percentage": (total_maintenance_cost / asset.gross_purchase_amount * 100) if asset.gross_purchase_amount else 0
        }
```

## Setup Module Patterns

### Configuration Management

```python
class ConfigurationManager:
    """Advanced configuration management with inheritance"""
    
    def __init__(self, company):
        self.company = company
        self.config_cache = {}
    
    def get_setting(self, setting_name, doctype="Company", fallback_chain=None):
        """Get setting with fallback hierarchy"""
        
        cache_key = f"{doctype}:{self.company}:{setting_name}"
        
        if cache_key in self.config_cache:
            return self.config_cache[cache_key]
        
        # Try company-specific setting first
        if doctype == "Company":
            value = frappe.db.get_value("Company", self.company, setting_name)
            if value is not None:
                self.config_cache[cache_key] = value
                return value
        
        # Try global settings
        if fallback_chain:
            for fallback_doctype in fallback_chain:
                value = frappe.db.get_single_value(fallback_doctype, setting_name)
                if value is not None:
                    self.config_cache[cache_key] = value
                    return value
        
        # Return default
        return None
    
    def setup_company_defaults(self, company_data):
        """Setup company with intelligent defaults"""
        
        # Create company
        company_doc = frappe.get_doc({
            "doctype": "Company",
            "company_name": company_data["name"],
            "country": company_data["country"],
            "default_currency": company_data.get("currency") or self.get_country_currency(company_data["country"])
        })
        
        company_doc.insert()
        
        # Setup default accounts based on country
        self.setup_default_accounts(company_doc, company_data["country"])
        
        # Setup default warehouses
        self.setup_default_warehouses(company_doc)
        
        # Setup default fiscal year
        self.setup_fiscal_year(company_doc, company_data["country"])
        
        return company_doc
    
    def setup_regional_customizations(self, company, country):
        """Apply country-specific customizations"""
        
        regional_settings = self.get_regional_settings(country)
        
        if regional_settings:
            # Apply tax settings
            if "taxes" in regional_settings:
                self.setup_tax_accounts(company, regional_settings["taxes"])
            
            # Apply reporting requirements
            if "reports" in regional_settings:
                self.enable_country_reports(company, regional_settings["reports"])
            
            # Apply compliance settings
            if "compliance" in regional_settings:
                self.setup_compliance_settings(company, regional_settings["compliance"])
```

## Module Extension Patterns

### Custom Module Development Framework

```python
class ModuleExtensionFramework:
    """Framework for creating custom modules that integrate with ERPNext"""
    
    def __init__(self, module_name):
        self.module_name = module_name
        self.hooks = []
        self.custom_doctypes = []
        self.custom_fields = []
    
    def register_hooks(self, hook_config):
        """Register module hooks for integration"""
        
        for hook_type, hook_methods in hook_config.items():
            if hook_type == "doc_events":
                for doctype, events in hook_methods.items():
                    for event, method in events.items():
                        self.hooks.append({
                            "type": "doc_event",
                            "doctype": doctype,
                            "event": event,
                            "method": f"{self.module_name}.{method}"
                        })
        
        return self.hooks
    
    def create_integration_points(self, erpnext_module):
        """Create integration points with existing ERPNext modules"""
        
        integration_map = {
            "accounts": self.integrate_with_accounts,
            "stock": self.integrate_with_stock,
            "selling": self.integrate_with_selling,
            "buying": self.integrate_with_buying,
            "manufacturing": self.integrate_with_manufacturing
        }
        
        if erpnext_module in integration_map:
            return integration_map[erpnext_module]()
    
    def integrate_with_accounts(self):
        """Integration patterns with Accounts module"""
        
        return {
            "gl_integration": {
                "custom_gl_entries": f"{self.module_name}.accounts.make_gl_entries",
                "payment_integration": f"{self.module_name}.accounts.process_payments"
            },
            "reports": {
                "custom_financial_reports": f"{self.module_name}.reports.financial"
            }
        }
    
    def integrate_with_stock(self):
        """Integration patterns with Stock module"""
        
        return {
            "stock_integration": {
                "custom_stock_entries": f"{self.module_name}.stock.make_stock_entries",
                "valuation_method": f"{self.module_name}.stock.custom_valuation"
            },
            "inventory_reports": {
                "custom_inventory_reports": f"{self.module_name}.reports.inventory"
            }
        }

class CustomizationManager:
    """Manage customizations without breaking core functionality"""
    
    def __init__(self):
        self.customizations = []
    
    def apply_safe_customization(self, doctype, customization_data):
        """Apply customizations safely with rollback capability"""
        
        try:
            # Create backup of current state
            backup = self.create_customization_backup(doctype)
            
            # Apply customization
            self.execute_customization(doctype, customization_data)
            
            # Validate customization
            validation_result = self.validate_customization(doctype)
            
            if not validation_result["success"]:
                # Rollback if validation fails
                self.rollback_customization(doctype, backup)
                return {
                    "success": False,
                    "error": validation_result["error"]
                }
            
            return {"success": True}
            
        except Exception as e:
            frappe.log_error("Customization failed", str(e))
            return {
                "success": False,
                "error": str(e)
            }
    
    def create_version_controlled_customization(self, module_name, customizations):
        """Create version-controlled customizations"""
        
        customization_doc = frappe.get_doc({
            "doctype": "Custom Module",
            "module_name": module_name,
            "version": "1.0.0",
            "customizations": json.dumps(customizations),
            "status": "Development"
        })
        
        customization_doc.insert()
        
        return customization_doc

class APIExtensionManager:
    """Manage API extensions and integrations"""
    
    def create_rest_api_endpoint(self, endpoint_config):
        """Create RESTful API endpoints for custom functionality"""
        
        @frappe.whitelist(allow_guest=False, methods=["GET", "POST"])
        def custom_api_handler():
            try:
                # Authentication and authorization
                self.validate_api_access()
                
                # Process request
                if frappe.request.method == "GET":
                    return self.handle_get_request(endpoint_config)
                elif frappe.request.method == "POST":
                    return self.handle_post_request(endpoint_config)
                
            except Exception as e:
                frappe.log_error("API Error", str(e))
                return {"error": str(e)}
        
        return custom_api_handler
    
    def create_webhook_integration(self, webhook_config):
        """Create webhook integrations for external systems"""
        
        webhook_doc = frappe.get_doc({
            "doctype": "Webhook",
            "webhook_doctype": webhook_config["doctype"],
            "webhook_docevent": webhook_config["event"],
            "request_url": webhook_config["url"],
            "request_structure": "Form URL-Encoded",
            "webhook_headers": webhook_config.get("headers", []),
            "webhook_data": webhook_config.get("data", []),
            "condition": webhook_config.get("condition", "")
        })
        
        webhook_doc.insert()
        
        return webhook_doc
```

## Conclusion

This comprehensive analysis of ERPNext's module-specific patterns provides deep insights into how different business domains are expertly handled through specialized patterns that have evolved from real-world enterprise requirements. Each module demonstrates sophisticated approaches to domain-specific challenges while maintaining seamless integration across the entire system.

### Key Takeaways

1. **Domain Expertise**: Each module encapsulates deep domain knowledge with patterns specific to business requirements
2. **Integration Excellence**: Cross-module integration is handled through well-defined interfaces and event-driven architecture
3. **Extensibility**: Robust patterns for customization and extension without breaking core functionality
4. **Performance Optimization**: Module-specific optimizations for high-volume operations
5. **Workflow Orchestration**: Sophisticated business process automation within and across modules
6. **Data Integrity**: Advanced validation and consistency patterns across complex business processes

These patterns serve as proven blueprints for building enterprise-grade applications on the Frappe framework, demonstrating how to handle complex business requirements while maintaining system integrity and performance.