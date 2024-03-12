# Postgres

Introduction
The vacuum process in PostgreSQL plays a critical role in maintaining database health by removing 'dead' tuples to prevent space bloat. Traditional vacuuming operates at the relation level, processing entire tables or indexes collectively. This project introduces a custom implementation, comprising custom_vacuum_page and perform_custom_vacuum functions, to enable more precise and efficient vacuuming at the individual page level.

Objective
The primary objective of this project is to augment PostgreSQL's vacuum capabilities, allowing for:
1. Page-Level Vacuuming: To target and process individual pages within a relation.
2. Selective Tuple Management: To identify and remove dead tuples efficiently at the page level.
3. Enhanced Vacuum Control: To offer database administrators the ability to apply vacuum processes in a more targeted manner.


Design and Implementation
custom_vacuum_page Function
This function is responsible for processing a single page within a relation. The implementation involves:
1. Buffer Management: Locking the buffer for exclusive access to ensure data integrity during processing.
2. Page-Level Tuple Processing: Iterating through items on the page, identifying and removing dead tuples.
3. Buffer Update and Release: Marking the buffer as 'dirty' after processing and releasing the lock.


perform_custom_vacuum Function
This function orchestrates the vacuum process across an entire relation by:
 Validating Relations: Checking the provided VacuumRelation object for operational validity.
 Iterating Over Blocks: Applying custom_vacuum_page logic to each block.
 Completing the Vacuum Process: Closing the relation post-processing.


Integration into PostgreSQL
The custom vacuum functions are integrated into the PostgreSQL source code, specifically within the vacuum.c file. They are invoked as part of the database's regular vacuuming process, ensuring compatibility and cohesion with the existing vacuum logic.




Test Cases
To evaluate the effectiveness of the custom vacuum functions, several test cases were designed, including:
1. High Transaction Environment: Simulating a database with frequent updates and deletions to assess the performance improvements.
2. Space Utilization: Comparing the space usage before and after applying the custom vacuum process.
3. Resource Utilization: Monitoring CPU and I/O usage during the vacuum process to measure efficiency gains.
![image](https://github.com/AkshaySudini/Postgres/assets/55602633/fd81a208-bb49-4386-8fb1-8d71ed0f5639)
