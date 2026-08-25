# AWS SysDE — Glassdoor Interview Questions (Organized)

Source: raw Glassdoor question dump, reorganized by topic, grammar cleaned up,
exact duplicates removed. Nothing has been cut for length or simplified — every
distinct question/prompt from the original list is preserved below, just sorted
into categories.

---

## 1. Behavioral / Amazon Leadership Principles

- Why Amazon?
- What is the coolest thing you've ever done?
- Tell me about a time you showed bias for action.
- Describe a time you had a disagreement with your boss and how you handled it.
- Tell me about a time you did something that had an unfavorable outcome.
- Name a situation where you pushed for something and it was rejected.
- Did you take an action without asking permission to fix an issue that impacted users?
- Tell me about a time you went above and beyond for a customer.
- Tell me about a time you had to make an unpopular decision, and what the outcome was.
- Tell me about a time you were wrong at work.
- Describe a time you innovated on a project and how it benefited your team.
- Tell me about a time you had to deliver on a deadline.
- Tell me about a time you took on something significant outside your area of responsibility — not something that would commonly fall within your day-to-day job responsibilities.
- What's the most innovative thing you've built?
- How do you handle a large task and make sure it's completed on time?
- Name a project or accomplishment you're proud of.
- Tell me about a big problem you had to manage and how you reacted.
- Describe a project.
- Tell me about a project you're proud of.
- Tell me about a big problem you solved.
- How do you know it's time to make a change on a team — personnel-wise, technically?
- Describe a time you bypassed your manager in making a business decision. What was your reaction to the most urgent problem?
- Some situational questions about hurdles or challenges you've overcome in life.
- Tell me about your background.
- Tell me about a distinct project.
- Describe a time a product delivery didn't happen properly, and the action you took.

---

## 2. Linux & OS Fundamentals

- Stuff about Linux commands, such as what `chmod` does, etc.
- What's in `/proc`?
- Linux distros, processes, memory management, and disk management.
- In-depth questions on Linux process and resource troubleshooting.
- What basic Linux tools have you used?
- You accidentally entered `cd /bin; chmod 644 chmod` — how do you fix it?
- Linux complex commands.
- Basic Linux internals, algorithmic and TCP/IP-related questions — e.g., how does traceroute work?
- Difference between a daemon process and a foreground process.
- How do you kill a zombie process?
- What is thrashing in an operating system?
- Discussion on operating systems and computer network concepts.
- Implement a priority scheduler for an OS.
- Linux internals: inode, OOM killer, disk space cleanup, `fstab`, swap space.
- What is swappiness?
- What are system load averages?
- CPU affinity?
- Memory management? Process management?
- What is the most complex problem you've solved (Linux/systems context)?

---

## 3. Networking

- What is the DNS flow?
- Describe how DNS works.
- What happens when you navigate to amazon.com? / What happens when you type "amazon" into a browser?
- Describe how a wireless router works at home.
- Troubleshooting exercise: a website is unavailable — describe the steps you'd take to troubleshoot (asked more than once, in slightly different forms).
- Describe the TCP/IP connection process.
- Describe HTTPS certificate trust and SSL encryption.
- Hashmap, arrays, TCP/IP (topics list from one interview).
- Second interview was entirely networking: TCP/IP, Linux, OS-level questions.
- What is a network gateway?
- What are subnets?
- What is a switch vs. a hub? A switch vs. a router?
- Different IP classes.
- MAC and IP troubleshooting.
- DNS, DHCP, and the DORA process.
- Public and private gateway.
- NAT.
- Difference between TCP and UDP.
- SSL — i.e., HTTP vs. HTTPS, and HTTP-to-HTTPS routing.

---

## 4. Security & Cryptography

- I had prepared deeply at the AWS system-design level, but the interview instead asked security basics: hashing, symmetric vs. asymmetric cryptography, three-tier architecture, synchronous vs. asynchronous. (This was flagged twice in the source data — clearly a recurring surprise for candidates who over-index on system design prep.)

---

## 5. SQL

- SQL queries involving joins.

---

## 6. System Design

- How would you design the infrastructure for a music-streaming app such as Spotify?
- Design a certificate distribution system.
- System design: design a URL-shortening service.
- System design question: design a parking lot. (One candidate answered by discussing a serverless architecture.)
- Design an elevator system — minimize stops, and make it efficient.
- How would you design an auto-suggestion search bar?
- What would you do if page speed is slow?
- Design a payment system similar to Redbus.
- Design a tool that collects logs from all servers in a region. Also: if an instance suddenly went missing, how would you go about figuring out the issue?
- Design the IRCTC system.
- Imagine Amazon's shopping website doesn't exist — how would you create the infrastructure to host it?
- The database service isn't scaling well — how would you improve it?
- How would you save a user's actions and cart?
- The website needs to serve users in different countries — how would you do that?
- How would you distinguish good users from attackers?
- You need to change a configuration file. How would you do that? How would you do it reliably across many servers? How would you roll it back?
- You're running a web application, and after 200 API calls you start getting 5xx and 4xx errors. How would you debug this?
- System design and data structures round also covered: reverse a linked list, find the middle of a linked list, remove duplicates from a file, DNS and IP.

---

## 7. Coding / Algorithms / LeetCode-style

- Similar to LeetCode 736, "Parse Lisp Expression."
- Can you first explain your brute-force solution, then optimize it step by step while discussing time and space complexity? (General interviewer prompt style, applies across coding rounds.)
- Python parsing and Linux fundamentals — more than 50% of one interview's time was spent here.
- Started with a sliding-window question, followed by a DP (dynamic programming) question; had to code in an online notepad reviewable by the interviewer; finished with a behavioral question from the Amazon Leadership Principles.
- Coding task related to a log file: extract data from the log file using Python.
- Coding task using Linux and Python: get a file from a Linux server and validate it against the current file you have.
- Build an interface for Linux `find` functionality — e.g., filters by size (less than 10MB, greater than 20MB) and by type (PNG files, TXT files).
- Coding exercise: write a function to read a log file and print basic statistics.
- Coding exercise: given an org chart, write a function to return the common manager of two employees.
- One coding question on the Euler totient function, plus several MCQs.
- Two technical rounds, each with two questions to solve — covering basic math, graphs, trees, and hashmaps.
- Process two log files and compute the average transaction size for each IP.
- Given a log with a specified format, write a script to find the IP that accumulated the biggest total size accessed.
- Write a Python function that takes a string and a list, then uses the list's values to split the string into another list.
- Had to write programs/pseudocode on the spot.
- Algorithm and data-structure-based questions.
- 1st round: problem solving — an easy graph question, and a palindrome string-creation question.
- Technical question plus one code challenge: web service/backend app (discussed load balancing); Unix task (find files with size greater than 200MB); coding task (given an integer array, move all 0s to the left end).
- Recursion over a directory tree, in Python.
- Given an input string like `"hello world today"`, output `"today world hello"` (reverse the word order).
- Coding question: find the user with the lowest latency from a log file, e.g.:
  ```
  200,John,/home,60ms
  200,Sarah,/log,13ms
  500,Jack,/home,40ms
  ```
- Some programming questions — e.g., how do you add the number 5 to a set of numbers (example: `235`) to produce the maximum possible output?
- Two programming tasks plus 15 MCQs.
- Programming question: given an array of arrays, flatten into a single array. Example: `[[1,2,3,4],[5,6],[7]]` → `[1,2,3,4,5,6,7]`. Once solved, the complexity was increased to arrays nested within each other (arbitrary depth).
- Coding: an array question, with a follow-up about its complexity.
- Data structures, Unix, SQL, Git — listed together as one round's topic set.
- Find the common elements between two given arrays.
- Find the numbers whose digits sum to an odd value, within a given range (1000–3000).

---

## 8. PLC / IEC 61131-3 (hardware-adjacent — worth noting since this is a Hardware Engineering role)

- Implement a PLC program to control a fridge compressor and an interior light, using three IEC 61131-3 implementation languages: CFC, LD, and ST.
- Draw a PLC diagram to find the biggest number among five numbers.

---

## 9. Interview process / round-structure notes (not questions themselves, but useful context)

- MCQ round on computer fundamentals, covering around 9 sections.
- There are two rounds, both technical; each round has two questions to solve.
- 14 Amazon Leadership Principles, system design, and algorithms were all covered across the loop.
- Heavy emphasis on behavioral questions centered on the Amazon Leadership Principles; several situational questions probing knowledge/experience in networking, Linux, system design, and coding.
- Round was fairly technical, touching on the TCP/IP stack, networking, Linux internals and commands, and a moderately difficult algorithmic question.
- Fourth section of one loop: 5 minutes given to ask the interviewer questions.
- Both interviews (in one candidate's loop) had repetitive behavioral questions.
- Second round: discussion of projects and core CS subjects.

---



# AWS SysDE — Log-File Related Questions (Grouped)

Every question from the original Glassdoor dump that involves parsing, processing,
or designing around log files, pulled out and grouped together as its own set.

---

## Coding tasks — parsing / processing a log file

- Phone screen consisted of 2 Leadership Principles questions, then a coding task related to a log file: extract data from the log file using Python.
- Coding exercise: write a function to read a log file and print basic statistics.
- Process two log files and compute the average transaction size for each IP.
- Given a log with a specified format, write a script to find the IP that accumulated the biggest total size accessed.
- Coding question: find the user with the lowest latency from a log file, e.g.:
  ```
  200,John,/home,60ms
  200,Sarah,/log,13ms
  500,Jack,/home,40ms
  ```

## System design — log infrastructure (related, but a design question rather than a coding task)

- A design question for a tool that can collect logs from all servers in a region. Also: if an instance suddenly went missing, how would you go about figuring out the issue?

---

## Why these are worth drilling as a set

All five coding tasks share the same shape: **read structured/semi-structured text
line by line, parse fields (often comma- or space-delimited), and compute some
aggregate** — a count, an average, a max, or a "find the extreme value" lookup,
usually grouped by a key like IP or user. That's a distinct skill from the 17
"design a class" LeetCode problems — it's closer to basic text parsing +
`collections.defaultdict` / `Counter` work than to OOP API design.

If you want, I can write out Python solutions for a representative version of
each of these five (a plausible log format + parser + aggregation), sized for
practicing in your remaining prep days.
