---
name: adhere-to-strict-rule-specifications
description: Use when completing tasks with specific schema, output naming, formatting, or organizational rules.
---
<body>
1. Read all instructions, prompt descriptions, and rule checklists completely before writing any code or data outputs.
2. Maintain a checklist of every explicit rule constraint (e.g., specific JSON keys, header names, exact casing/transformations like replacing hyphens with underscores, integer cents for monetary values, schema versions, or mandatory metadata blocks).
3. Implement data formatting transformations explicitly according to rules (e.g., converting floating-point monetary values to integer cents by multiplying and rounding).
4. Verify all output files against every requirement in the rule checklist using automated assertions or inspection scripts before finishing.
5. Self-Check: Did I include all required keys/headers, exact naming conventions, and proper data types/units specified by the rules?
