Perfect! Let's focus on the core enterprise need - schemas and objects. Here's a streamlined approach targeting exactly what you need:
```python
import re
from dataclasses import dataclass, field
from typing import List, Dict, Set, Optional, Tuple
from enum import Enum
from antlr4 import *
from PlsqlLexer import PlsqlLexer
from PlsqlParser import PlsqlParser
from PlsqlParserBaseVisitor import PlsqlParserBaseVisitor

class RelationshipType(Enum):
    CALLS = "CALLS"           # SP calls another SP
    READS = "READS"           # SELECT from table/view
    WRITES = "WRITES"         # INSERT/UPDATE/DELETE on table
    CREATES = "CREATES"       # CREATE TABLE (temp tables)
    REFERENCES = "REFERENCES" # Generic reference (unclear operation)

class ObjectType(Enum):
    PROCEDURE = "PROCEDURE"
    FUNCTION = "FUNCTION"
    TABLE = "TABLE"
    VIEW = "VIEW"
    TEMP_TABLE = "TEMP_TABLE"

@dataclass
class SchemaObject:
    """Represents any database object with proper schema handling"""
    name: str
    schema: Optional[str] = None
    object_type: ObjectType = ObjectType.TABLE
    
    @property
    def full_name(self) -> str:
        """Returns schema.object_name or just object_name"""
        return f"{self.schema}.{self.name}" if self.schema else self.name
    
    @property 
    def unique_id(self) -> str:
        """Returns unique identifier for graph storage"""
        schema_part = self.schema or "unknown"
        return f"{schema_part}.{self.name}.{self.object_type.value}"

@dataclass
class ObjectDependency:
    """Represents a dependency between two database objects"""
    source_object: SchemaObject      # The SP that's doing the operation
    target_object: SchemaObject      # The object being accessed
    relationship: RelationshipType   # What type of operation
    line_number: int                # Where in the code
    sql_snippet: str = ""           # The actual SQL code
    is_ambiguous: bool = False      # True if schema resolution uncertain

class SchemaObjectExtractor(PlsqlParserBaseVisitor):
    """
    Focused extractor for enterprise schema and object dependencies
    """
    
    def __init__(self, default_schemas: List[str] = None):
        # Configuration
        self.default_schemas = default_schemas or ["dbo", "sales", "hr", "finance"]
        self.known_schemas = set(self.default_schemas)  # Will be populated from actual DB
        
        # State tracking
        self.current_procedure: Optional[SchemaObject] = None
        self.dependencies: List[ObjectDependency] = []
        self.temp_objects: Set[str] = set()  # Track temp tables in current SP
        
        # Schema resolution cache
        self.schema_cache: Dict[str, List[str]] = {}  # object_name -> [possible_schemas]
    
    def add_known_schemas(self, schemas: List[str]):
        """Add known schemas from database metadata"""
        self.known_schemas.update(schemas)
    
    def add_schema_objects(self, schema_objects: Dict[str, List[str]]):
        """
        Add known objects per schema for better resolution
        schema_objects = {"sales": ["Customer", "Order"], "hr": ["Employee"]}
        """
        for schema, objects in schema_objects.items():
            for obj in objects:
                if obj not in self.schema_cache:
                    self.schema_cache[obj] = []
                self.schema_cache[obj].append(schema)
    
    # =============================================================================
    # CORE OBJECT NAME EXTRACTION
    # =============================================================================
    
    def extract_object_name(self, ctx) -> Tuple[str, Optional[str]]:
        """
        Extract object name and schema from various PLSQL contexts
        Returns: (object_name, schema_name_or_none)
        """
        if not ctx:
            return "", None
            
        text = ctx.getText()
        
        # Handle schema.object pattern
        if '.' in text:
            parts = text.split('.')
            if len(parts) == 2:
                schema, obj_name = parts[0], parts[1]
                return obj_name.strip('"'), schema.strip('"')
        
        # Just object name, no schema
        return text.strip('"'), None
    
    def resolve_schema(self, object_name: str, provided_schema: Optional[str]) -> Tuple[str, bool]:
        """
        Resolve the actual schema for an object
        Returns: (resolved_schema, is_ambiguous)
        """
        if provided_schema:
            return provided_schema, False
        
        # Check cache for known locations of this object
        if object_name in self.schema_cache:
            possible_schemas = self.schema_cache[object_name]
            if len(possible_schemas) == 1:
                return possible_schemas[0], False
            elif len(possible_schemas) > 1:
                # Multiple schemas have this object - ambiguous!
                return possible_schemas[0], True  # Pick first but mark ambiguous
        
        # Default to first known schema
        return self.default_schemas[0], True
    
    def create_schema_object(self, name: str, schema: Optional[str], 
                           obj_type: ObjectType) -> Tuple[SchemaObject, bool]:
        """Create SchemaObject with proper resolution"""
        resolved_schema, is_ambiguous = self.resolve_schema(name, schema)
        
        # Handle temp tables (scope to current procedure)
        if name.startswith('#') and self.current_procedure:
            resolved_schema = f"temp_{self.current_procedure.name}"
            obj_type = ObjectType.TEMP_TABLE
            is_ambiguous = False
        
        obj = SchemaObject(name=name, schema=resolved_schema, object_type=obj_type)
        return obj, is_ambiguous
    
    def add_dependency(self, target_name: str, target_schema: Optional[str], 
                      relationship: RelationshipType, ctx, 
                      target_type: ObjectType = ObjectType.TABLE):
        """Add a dependency to our collection"""
        if not self.current_procedure:
            return
        
        target_obj, is_ambiguous = self.create_schema_object(
            target_name, target_schema, target_type)
        
        dependency = ObjectDependency(
            source_object=self.current_procedure,
            target_object=target_obj,
            relationship=relationship,
            line_number=ctx.start.line if ctx.start else 0,
            sql_snippet=ctx.getText()[:100] if ctx else "",
            is_ambiguous=is_ambiguous
        )
        
        self.dependencies.append(dependency)

    # =============================================================================
    # PROCEDURE/FUNCTION DEFINITIONS  
    # =============================================================================
    
    def visitCreate_procedure(self, ctx: PlsqlParser.Create_procedureContext):
        """Handle CREATE PROCEDURE statements"""
        if hasattr(ctx, 'procedure_name') and ctx.procedure_name():
            name, schema = self.extract_object_name(ctx.procedure_name())
            self.current_procedure, _ = self.create_schema_object(
                name, schema, ObjectType.PROCEDURE)
        
        return self.visitChildren(ctx)
    
    def visitCreate_function(self, ctx: PlsqlParser.Create_functionContext):
        """Handle CREATE FUNCTION statements"""
        if hasattr(ctx, 'function_name') and ctx.function_name():
            name, schema = self.extract_object_name(ctx.function_name())
            self.current_procedure, _ = self.create_schema_object(
                name, schema, ObjectType.FUNCTION)
        
        return self.visitChildren(ctx)

    # =============================================================================
    # PROCEDURE CALLS (EXEC statements)
    # =============================================================================
    
    def visitExecute_immediate(self, ctx: PlsqlParser.Execute_immediateContext):
        """Handle EXECUTE IMMEDIATE (dynamic SQL)"""
        # Mark as dynamic reference for now
        if hasattr(ctx, 'expression') and ctx.expression():
            self.add_dependency("DYNAMIC_SQL", None, RelationshipType.REFERENCES, ctx)
        return self.visitChildren(ctx)
    
    def visitProcedure_call(self, ctx):
        """Handle direct procedure calls"""
        # This depends on your grammar - adjust the attribute names
        if hasattr(ctx, 'routine_name') and ctx.routine_name():
            name, schema = self.extract_object_name(ctx.routine_name())
            self.add_dependency(name, schema, RelationshipType.CALLS, ctx, ObjectType.PROCEDURE)
        
        return self.visitChildren(ctx)

    # =============================================================================
    # TABLE OPERATIONS (SELECT, INSERT, UPDATE, DELETE)
    # =============================================================================
    
    def visitSelect_statement(self, ctx: PlsqlParser.Select_statementContext):
        """Handle SELECT statements"""
        self.extract_table_references(ctx, RelationshipType.READS)
        return self.visitChildren(ctx)
    
    def visitInsert_statement(self, ctx: PlsqlParser.Insert_statementContext):
        """Handle INSERT statements"""
        self.extract_table_references(ctx, RelationshipType.WRITES)
        return self.visitChildren(ctx)
    
    def visitUpdate_statement(self, ctx: PlsqlParser.Update_statementContext):
        """Handle UPDATE statements"""
        self.extract_table_references(ctx, RelationshipType.WRITES)
        return self.visitChildren(ctx)
    
    def visitDelete_statement(self, ctx: PlsqlParser.Delete_statementContext):
        """Handle DELETE statements"""
        self.extract_table_references(ctx, RelationshipType.WRITES)
        return self.visitChildren(ctx)
    
    def extract_table_references(self, ctx, relationship: RelationshipType):
        """
        Generic method to extract table references from SQL statements
        You'll need to adapt this based on your specific grammar structure
        """
        # This is a simplified approach - you'll need to traverse the specific
        # grammar rules for table references in your PLsql grammar
        
        # Look for table_name contexts (adjust based on your grammar)
        if hasattr(ctx, 'table_name'):
            for table_ctx in self.find_all_table_names(ctx):
                name, schema = self.extract_object_name(table_ctx)
                self.add_dependency(name, schema, relationship, table_ctx)
    
    def find_all_table_names(self, ctx):
        """
        Recursively find all table name contexts
        This needs to be adapted to your specific grammar rules
        """
        table_names = []
        
        # Traverse all children looking for table references
        if hasattr(ctx, 'children') and ctx.children:
            for child in ctx.children:
                if hasattr(child, 'getText'):
                    # Check if this looks like a table reference
                    # This is grammar-specific logic
                    if self.looks_like_table_reference(child):
                        table_names.append(child)
                
                # Recursively check children
                table_names.extend(self.find_all_table_names(child))
        
        return table_names
    
    def looks_like_table_reference(self, ctx) -> bool:
        """
        Heuristic to identify table references
        Adapt this based on your grammar structure
        """
        if not hasattr(ctx, 'getText'):
            return False
            
        text = ctx.getText()
        
        # Skip if it looks like a column name or function call
        if '(' in text or text.isupper() and len(text) < 3:
            return False
            
        # Skip common SQL keywords
        sql_keywords = {'SELECT', 'FROM', 'WHERE', 'AND', 'OR', 'AS', 'ON'}
        if text.upper() in sql_keywords:
            return False
            
        return True

    # =============================================================================
    # CREATE TABLE (for temp tables)
    # =============================================================================
    
    def visitCreate_table(self, ctx: PlsqlParser.Create_tableContext):
        """Handle CREATE TABLE statements"""
        if hasattr(ctx, 'table_name') and ctx.table_name():
            name, schema = self.extract_object_name(ctx.table_name())
            
            # Determine if it's a temp table
            obj_type = ObjectType.TEMP_TABLE if name.startswith('#') else ObjectType.TABLE
            
            self.add_dependency(name, schema, RelationshipType.CREATES, ctx, obj_type)
            
            if name.startswith('#'):
                self.temp_objects.add(name)
        
        return self.visitChildren(ctx)

# =============================================================================
# MAIN PROCESSING FUNCTION
# =============================================================================

def extract_dependencies_from_sql(sql_content: str, 
                                 known_schemas: List[str] = None,
                                 schema_objects: Dict[str, List[str]] = None) -> List[ObjectDependency]:
    """
    Main function to extract dependencies from SQL content
    
    Args:
        sql_content: The PLSQL procedure content
        known_schemas: List of known schema names
        schema_objects: Dict mapping schemas to their objects
    
    Returns:
        List of ObjectDependency objects
    """
    
    # Parse SQL using ANTLR
    input_stream = InputStream(sql_content)
    lexer = PlsqlLexer(input_stream)
    token_stream = CommonTokenStream(lexer)
    parser = PlsqlParser(token_stream)
    
    # Get the parse tree (adjust root rule based on your grammar)
    tree = parser.sql_script()  # or whatever your root rule is
    
    # Create extractor and configure it
    extractor = SchemaObjectExtractor(known_schemas)
    
    if schema_objects:
        extractor.add_schema_objects(schema_objects)
    
    # Walk the tree
    extractor.visit(tree)
    
    return extractor.dependencies

# =============================================================================
# EXAMPLE USAGE
# =============================================================================

if __name__ == "__main__":
    # Example SQL content
    sql_sample = """
    CREATE PROCEDURE sales.usp_ProcessOrder(@orderId INT)
    AS
    BEGIN
        SELECT * FROM sales.Orders WHERE id = @orderId;
        INSERT INTO billing.Invoice (order_id, amount) 
        SELECT id, total FROM sales.Orders WHERE id = @orderId;
        
        EXEC finance.usp_CalculateTax @orderId;
        
        CREATE TABLE #TempResults (id INT, name VARCHAR(50));
        INSERT INTO #TempResults SELECT id, customer_name FROM Customer;
    END
    """
    
    # Configure known schemas and objects
    known_schemas = ["sales", "billing", "finance", "hr", "dbo"]
    schema_objects = {
        "sales": ["Orders", "Customer", "Product"],
        "billing": ["Invoice", "Payment"],
        "finance": ["TaxRate", "Account"]
    }
    
    # Extract dependencies
    dependencies = extract_dependencies_from_sql(
        sql_sample, known_schemas, schema_objects)
    
    # Print results
    for dep in dependencies:
        ambiguous = " (AMBIGUOUS)" if dep.is_ambiguous else ""
        print(f"{dep.source_object.full_name} -> {dep.relationship.value} -> {dep.target_object.full_name}{ambiguous}")
```

This schema-focused approach gives you:

## 🎯 **Key Features**

**1. Smart Schema Resolution**
- Handles `schema.object` and bare `object` names
- Uses known schema mappings to resolve ambiguous references
- Flags when multiple schemas contain the same object name

**2. Enterprise Object Types**
- Procedures, Functions, Tables, Views, Temp Tables
- Proper scoping for temp tables (tied to parent procedure)

**3. Relationship Types**
- `CALLS`: SP → SP calls  
- `READS`: SP → Table (SELECT operations)
- `WRITES`: SP → Table (INSERT/UPDATE/DELETE)
- `CREATES`: SP → Table (CREATE TABLE, especially temp tables)

**4. Ambiguity Handling**
- Marks dependencies as ambiguous when schema resolution is uncertain
- Provides fallback resolution strategies

## 🔧 **Customization Points**

You'll need to adapt the visitor methods based on your specific PLsql grammar structure:

1. **Grammar Rule Names**: Adjust `visitCreate_procedure`, `visitSelect_statement`, etc. to match your grammar
2. **Context Attributes**: Change `ctx.table_name()`, `ctx.procedure_name()` to match your parse tree structure  
3. **Table Reference Logic**: The `find_all_table_names()` method needs to be tailored to how your grammar represents table references

## 🚀 **Usage**

```python
# Configure your enterprise schemas
known_schemas = ["sales", "billing", "finance", "hr", "dbo"]
schema_objects = {
    "sales": ["Customer", "Order", "Product"],
    "billing": ["Invoice", "Payment"], 
    # ... populate from your actual database metadata
}

# Extract dependencies
deps = extract_dependencies_from_sql(procedure_content, known_schemas, schema_objects)

# Get clean results
for dep in deps:
    print(f"{dep.source_object.full_name} -> {dep.relationship.value} -> {dep.target_object.full_name}")
```

Want me to help you adapt the visitor methods to your specific PLsql grammar structure? Just show me a few examples of the parse tree context names from your grammar file.
