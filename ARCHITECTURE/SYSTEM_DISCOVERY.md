# 03_dependency_mapping.md – Dependency & Flow Mapping Prompt

Role: Act as a System Integration Architect.

Objective:
Build a logical dependency and interaction model of the legacy system.

Input:
- List of programs/modules
- Database tables
- Interfaces

Tasks:
1. Map:
   - Program → Program dependencies
   - Program → Database dependencies
   - Program → External system dependencies
2. Categorize coupling:
   - Strong
   - Medium
   - Weak
3. Identify:
   - Critical paths
   - High-risk components
   - Natural service boundaries

Output Format:
- Dependency Graph (textual)
- Critical Flow Paths
- Coupling Analysis
- Candidate Microservice Boundaries
- Refactoring Opportunities

Tone:
Architectural, structured.
