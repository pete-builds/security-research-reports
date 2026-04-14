---
title: "Effectiveness of Exam Software in T14 Law Schools for Detecting Academic Integrity Violations"
slug: exam-software-t14-law-schools-academic-integrity
date: 2026-03-26
summary: "Exam proctoring software used in U.S. law schools, dominated by ExamSoft/Examplify and Exam4, suffers from high false-positive rates, documented racial and disability bias, and security vulnerabilities that researchers have shown can be 'trivially bypassed.'"
---

## Findings

### 1. Software Landscape in Law Schools

Four exam software suites dominate U.S. legal education: **Examplify** (by ExamSoft/Turnitin), **ILG Exam360**, **Exam4** (by Extegrity), and **Electronic Blue Book**. Together these cover 93% of U.S. law schools and 100% of remote state bar exams, with Examplify holding the largest market share. ([Burgess et al., USENIX Security 2022](https://arxiv.org/abs/2205.03009))

These tools generally operate in two modes:

- **Lockdown mode**: The software takes control of the student's computer, disabling other applications, internet access, screenshots, and clipboard functions. This is the primary mode for in-person exams. Both [ExamSoft Examplify](https://examsoft.com/programs/law/) and [Exam4](https://exam4.com/) operate this way.
- **Remote proctoring mode**: For remote exams, additional monitoring layers are added. ExamSoft's **ExamMonitor** records audio and video via webcam, uses AI to detect "anomalies" and "unusual behaviors," and routes flagged incidents through a three-tier review process: automated AI review, human proctor review, and institutional administrator review. ExamSoft explicitly states that "flagging is not cheating" and that the company "never makes denunciations about integrity breaches," leaving final determinations to institutions. ([ExamSoft: Flagging Isn't Cheating](https://examsoft.com/resources/flagging-isnt-cheating/))

**Regarding T14 schools specifically**: Forum posts from [Top Law Schools](https://www.top-law-schools.com/forums/viewtopic.php?f=4&t=185191&start=25) indicate that multiple T14 schools use ExamSoft, with some requiring students to print outlines for open-book exams rather than allowing digital access. I was unable to compile a verified, comprehensive list of which T14 school uses which product.

### 2. False Positive Rates: The Core Problem

The most damning evidence against exam proctoring software effectiveness comes from documented false positive rates.

**California Bar Exam (October 2020)**: ExamSoft flagged approximately one-third of roughly 9,000 online examinees (about 3,190 people) for potential cheating. After human review, **98% of those flagged were cleared of misconduct**. Only 47 test-takers were ultimately implicated. ([EFF: ExamSoft Flags One-Third of Bar Exam Test Takers](https://www.eff.org/deeplinks/2020/12/examsoft-flags-one-third-california-bar-exam-test-takers-cheating))

This means the system's positive predictive value was roughly 1.5%. For every student correctly flagged, approximately 67 were falsely flagged.

**Faculty Review Gap**: ProctorU revealed that "only about 10 percent of faculty members review the video" for flagged students. A University of Iowa audit found similar results with Proctorio: only 14% of faculty analyzed flagged results. ([EFF: A Long Overdue Reckoning For Online Proctoring Companies](https://www.eff.org/deeplinks/2021/06/long-overdue-reckoning-online-proctoring-companies-may-finally-be-here))

**Proctoring companies themselves acknowledge the problem**: The academic literature notes that "OP company websites rarely if ever cite rigorous studies to justify their claims and to eliminate concerns about false positives." ([Selwyn et al., 2021, PMC](https://pmc.ncbi.nlm.nih.gov/articles/PMC8407138/))

### 3. Racial and Demographic Bias

**Facial detection failures by skin tone**: A 2022 peer-reviewed study in Frontiers in Education examined Respondus Monitor (used by 1,500+ universities) and found that students with darker skin tones showed "a significant increase in likelihood" of being flagged for potential cheating. Women with the darkest skin tones were "far more likely than darker skin males or lighter skin males and females to be flagged for review." ([Yoder-Himes et al., Frontiers in Education, 2022](https://www.frontiersin.org/journals/education/articles/10.3389/feduc.2022.881449/full))

**Proctorio's facial detection** fails to recognize Black faces more than 50 percent of the time. ([EFF: A Long Overdue Reckoning](https://www.eff.org/deeplinks/2021/06/long-overdue-reckoning-online-proctoring-companies-may-finally-be-here))

**ExamSoft and the California Bar**: The EFF reported that ExamSoft's flags for "leaving the view of the webcam" likely reflect facial recognition failures, which are "more likely to occur to Black and Brown students." ([EFF: ExamSoft Flags One-Third](https://www.eff.org/deeplinks/2020/12/examsoft-flags-one-third-california-bar-exam-test-takers-cheating))

**Documented student experiences**: A Black woman at CU Denver reported that Proctorio "always prompted her to shine more light on her face" and repeatedly denied her test access, while white peers faced no such issues. This was cited in a letter by six U.S. senators to ExamSoft's CEO. ([Auraria Library / Swauger, 2021](https://library.auraria.edu/news/2021/why-online-test-proctoring-biased-expert))

The USENIX study specifically evaluated facial recognition classifiers used by Examplify for potential racial bias in false cheating flags. ([Burgess et al., USENIX Security 2022](https://arxiv.org/abs/2205.03009))

### 4. Disability Discrimination

A 2025 study documented how proctoring systems disproportionately harm disabled students ([arXiv, November 2025](https://arxiv.org/html/2511.10826v1)):

- **Accommodation interference**: Systems actively blocked accessibility tools such as screen readers, magnification software, and dictation programs. One student was "ejected from a test for using a zooming function" needed for a visual disability.
- **Neurodivergent behavior flagging**: Stimming, fidgeting, and postural shifts necessary for some students with ADHD or other conditions were flagged as suspicious behavior.
- **Cognitive overload**: Students reported self-regulating their movements to avoid flags, reducing mental resources available for the exam itself.
- **Anxiety amplification**: Students with anxiety disorders reported that constant camera surveillance "increase[d] the anxiety levels," directly affecting test performance.
- **Extra time cutoffs**: Systems sometimes cut off allotted extra-time accommodations.

A 2025 law review article in the Journal of Legal Education specifically examined "How Remote Proctoring Continues to Discriminate Against Disabled Students" in legal education contexts. ([Journal of Legal Education, May 2025](https://scholarcommons.sc.edu/cgi/viewcontent.cgi?article=3248&context=jled))

### 5. Security Vulnerabilities and Evasion Methods

#### 5a. Fundamental Security Weaknesses

The USENIX Security 2022 study reverse-engineered all four major law school exam suites and concluded that **"all their anti-cheating measures can be trivially bypassed."** The researchers modeled three adversary levels: an ordinary law student, a law student with computer science experience, and an experienced reverse engineer. The software installed "highly privileged system services with full access to user activities," which themselves posed security risks to test-takers. ([Burgess et al., USENIX Security 2022](https://arxiv.org/abs/2205.03009))

#### 5b. Documented Evasion Categories

A 2024 ACM CCS study analyzed 137 social media videos and 4,297 comments on YouTube and TikTok, identifying both non-technical and deeply technical evasion methods ([Simko et al., ACM CCS 2024](https://seclab.cs.georgetown.edu/papers/simkoModernProblemsRequire2024.pdf)):

**Non-technical methods:**
- Sticky notes placed around webcam or monitor edges
- Notes written on clear plastic sheets placed over the laptop screen, allowing students to view exam content and notes simultaneously, undetectable to the outward-facing webcam
- Physical notes or textbooks positioned between eyes and screen while maintaining apparent "eye contact"
- Secondary devices (phones, tablets) positioned below webcam line of sight

**Hardware-based methods:**
- HDMI splitters/distribution amplifiers that split video signals to additional monitors while appearing as a single output to the computer
- External keyboards or mice positioned outside webcam view
- Wireless earpieces receiving answers from remote assistants

**Software-based methods:**
- Virtual machine exploitation: running proctoring software inside a VM while the host system runs other applications. "The VM's operating system continues to report that its window has focus" even when the host machine is active ([Binstein: How to Bypass Proctortrack](https://jakebinstein.com/blog/on-knuckle-scanners-and-cheating-how-to-bypass-proctortrack/))
- Webcam feed interception using tools like ManyCam to substitute pre-recorded video for live webcam feeds ([Binstein](https://jakebinstein.com/blog/on-knuckle-scanners-and-cheating-how-to-bypass-proctortrack/))
- Pre-recorded video routing to simulate legitimate exam-taking behavior

**Detection gap**: While some newer proctoring software can check if it is running in a VM, hardware-based approaches like HDMI splitters remain fundamentally undetectable because "the computer would 'see' only the splitter itself." ([Binstein](https://jakebinstein.com/blog/on-knuckle-scanners-and-cheating-how-to-bypass-proctortrack/))

The Simko et al. study also found that many students view proctoring software as "invasive surveillance technology" and share evasion methods with similar motivations to hacker/tinkerer communities. ([Simko et al., ACM CCS 2024](https://seclab.cs.georgetown.edu/papers/simkoModernProblemsRequire2024.pdf))

#### 5c. The Second-Device Problem

The most straightforward and widespread evasion method remains using a second device (phone, tablet, or laptop) that is entirely outside the proctoring software's control. Lockdown browsers can only control the device they are installed on. For take-home or remote exams, there is no technical mechanism to prevent a student from looking up answers on a separate device, short of continuous 360-degree room monitoring, which introduces its own privacy and feasibility problems.

### 6. Cost vs. Value

The California State Bar paid ExamSoft a five-year, $4 million contract. A state auditor's report questioned whether this expenditure was justified given the high false-positive rate. The EFF commented: "One has to wonder what, exactly, ExamSoft is offering that's worth $4 million given this high false-positive rate." ([EFF: A Long Overdue Reckoning](https://www.eff.org/deeplinks/2021/06/long-overdue-reckoning-online-proctoring-companies-may-finally-be-here))

### 7. Recent Developments (2025-2026)

The California Bar Exam has continued to face problems with exam technology:

- In February 2025, the bar exam experienced significant issues including the use of AI-generated questions and technical glitches, leading the Board of Trustees to order an independent investigation. ([California State Bar, March 2025](https://www.calbar.ca.gov/news/board-trustees-orders-independent-investigation-february-2025-bar-exam-issues))
- The State Bar of California admitted in April 2025 that it used AI to develop exam questions, compounding existing trust issues with exam technology. ([Los Angeles Times, April 2025](https://www.latimes.com/california/story/2025-04-23/state-bar-of-california-used-ai-for-exam-questions))
- As of March 2026, the exam vendor (Meazure Learning) was reported to have known it was unprepared for the February 2025 exam. ([Daily Journal, September 2025](https://www.dailyjournal.com/article/387746-ai-technological-cashouts-and-exam-reform-inside-the-california-bar-exam-crisis))

### 8. The Deterrence Question

A key unresolved question is whether exam software's primary value is detection or deterrence. Even if the software catches relatively few actual cheaters (given the 1.5% positive predictive value in the California Bar example), proponents argue it deters cheating by creating perceived risk. However:

- The existence of active online communities sharing evasion techniques (137 videos, 4,297 comments analyzed in just one study) suggests the deterrence effect may be limited among motivated students. ([Simko et al., ACM CCS 2024](https://seclab.cs.georgetown.edu/papers/simkoModernProblemsRequire2024.pdf))
- The Simko et al. research framed this as the development of a "security mindset" among students, suggesting evasion knowledge is spreading, not contracting.
- For in-person exams using lockdown-only software (no webcam), the software's primary function is preventing access to other applications on the same device, but it cannot prevent use of separate devices or physical notes.

## Confidence & Gaps

### High Confidence
- Four software suites dominate U.S. law school exams, covering 93% of schools (USENIX peer-reviewed)
- ExamSoft flagged ~33% of California Bar test-takers with ~98% false-positive clearance rate (EFF reporting on public records)
- All major anti-cheating measures can be "trivially bypassed" (USENIX peer-reviewed security research)
- Documented racial bias in facial detection, particularly affecting Black women (Frontiers in Education peer-reviewed)
- Disabled students face accommodation interference and disproportionate flagging (arXiv preprint + Journal of Legal Education)

### Medium Confidence
- ExamSoft/Examplify holds the largest market share specifically among T14 schools (inferred from overall dominance, no T14-specific survey found)
- The deterrence effect is limited among motivated students (reasonable inference from evasion community scale, no controlled study)
- Faculty non-review of flags (~10-14%) is representative across institutions (based on two data points)

### Gaps
- **No T14-specific adoption survey**: Could not find a verified list of which T14 schools use which product
- **No controlled effectiveness studies**: No peer-reviewed study has measured actual deterrence by comparing cheating rates with and without the software
- **Limited data on in-person lockdown mode**: Most research focuses on remote proctoring with webcam monitoring
- **No longitudinal data**: Whether evasion techniques are becoming more or less prevalent is not tracked systematically
- **AI-era gap**: How exam software handles AI-assisted cheating (LLMs on secondary devices) has minimal published research
