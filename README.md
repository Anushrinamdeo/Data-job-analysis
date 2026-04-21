# Decoding the Data Job Market

The job market for data analysts is noisy. Salary ranges are wide, skill requirements vary wildly, and it's genuinely hard to know where to focus your time. So I stopped guessing and started querying.

This project is a structured SQL investigation into the 2023 data analyst job market, cutting through the noise to answer one core question: **what does it actually take to land a well-paying, in-demand data analyst role?** The findings are sharper than I expected.

> SQL queries used throughout this project: [`/project_sql`](/project_sql/)

---

## The Investigation

I set out to answer five questions that matter to any analyst at any stage of their career:

1. Which data analyst roles pay the most, and who's hiring?
2. What skills do those top-paying roles demand?
3. Which skills appear most frequently across all job postings?
4. Which individual skills command the highest average salaries?
5. Where do high demand and high salary intersect the "optimal" skills?

Each question builds on the last. Together, they paint a clear picture of where the real value is in this market.

---

## Stack

| Tool | Role |
|------|------|
| **SQL** | Core analysis engine |
| **PostgreSQL** | Database management |
| **VS Code** | Query development environment |
| **Git & GitHub** | Version control and publishing |

---

## The Findings

### 1. The Top-Paying Roles

Starting with the ceiling. Filtering for remote Data Analyst roles with reported salaries, sorted by compensation:

```sql
SELECT	
    job_id,
    job_title,
    job_location,
    job_schedule_type,
    salary_year_avg,
    job_posted_date,
    name AS company_name
FROM job_postings_fact
LEFT JOIN company_dim ON job_postings_fact.company_id = company_dim.company_id
WHERE
    job_title_short = 'Data Analyst'
    AND job_location = 'Anywhere'
    AND salary_year_avg IS NOT NULL
ORDER BY salary_year_avg DESC
LIMIT 10;
```

**What the data showed:**

The salary range across the top 10 roles ran from **$184,000 to $650,000** a spread that reflects just how different "data analyst" can mean depending on seniority, scope, and company. Employers ranged from SmartAsset to Meta to AT&T, signaling that demand isn't concentrated in one sector. And the titles themselves, from Data Analyst to Director of Analytics, confirm there's a clear progression path with real financial upside.

![Top Paying Roles](assets/1_top_paying_roles.png)
*Salary breakdown for the top 10 highest-paying remote data analyst roles in 2023*

---

### 2. What Skills Those Top Roles Actually Require

High salary data is only useful if you know what skills earn it. This query joins job postings to skills data to reveal what top-paying employers are looking for:

```sql
WITH top_paying_jobs AS (
    SELECT	
        job_id,
        job_title,
        salary_year_avg,
        name AS company_name
    FROM job_postings_fact
    LEFT JOIN company_dim ON job_postings_fact.company_id = company_dim.company_id
    WHERE
        job_title_short = 'Data Analyst'
        AND job_location = 'Anywhere'
        AND salary_year_avg IS NOT NULL
    ORDER BY salary_year_avg DESC
    LIMIT 10
)

SELECT 
    top_paying_jobs.*,
    skills
FROM top_paying_jobs
INNER JOIN skills_job_dim ON top_paying_jobs.job_id = skills_job_dim.job_id
INNER JOIN skills_dim ON skills_job_dim.skill_id = skills_dim.skill_id
ORDER BY salary_year_avg DESC;
```

**Skill frequency across the top 10 roles:**

- **SQL** - 8 out of 10 roles
- **Python** - 7 out of 10 roles
- **Tableau** - 6 out of 10 roles
- **R**, **Snowflake**, **Pandas**, and **Excel** appeared with meaningful frequency as well

The pattern is clear: high-paying roles aren't looking for specialists in obscure tools. They want analysts who are strong in the fundamentals of SQL and Python, and can translate data into decisions using visualization tools like Tableau.

![Top Paying Skills](assets/2_top_paying_roles_skills.png)
*Skill frequency across the top 10 highest-compensated data analyst roles*

---

### 3. The Most In-Demand Skills Across All Postings

Top-paying roles are one slice. This query widens the lens to capture what employers are asking for across the full market:

```sql
SELECT 
    skills,
    COUNT(skills_job_dim.job_id) AS demand_count
FROM job_postings_fact
INNER JOIN skills_job_dim ON job_postings_fact.job_id = skills_job_dim.job_id
INNER JOIN skills_dim ON skills_job_dim.skill_id = skills_dim.skill_id
WHERE
    job_title_short = 'Data Analyst'
    AND job_work_from_home = True
GROUP BY skills
ORDER BY demand_count DESC
LIMIT 5;
```

| Skill | Demand Count |
|-------|-------------|
| SQL | 7,291 |
| Excel | 4,611 |
| Python | 4,330 |
| Tableau | 3,745 |
| Power BI | 2,609 |

**The takeaway:** SQL doesn't just top the salary charts it tops the volume charts too. And Excel's second-place ranking is a reminder that foundational business tools still carry serious weight. Python, Tableau, and Power BI complete a stack that covers querying, visualization, and storytelling which is exactly what hiring managers are looking for.

---

### 4. Which Skills Command the Highest Salaries

Demand tells you what's common. Salary tells you what's valued. This query finds the latter:

```sql
SELECT 
    skills,
    ROUND(AVG(salary_year_avg), 0) AS avg_salary
FROM job_postings_fact
INNER JOIN skills_job_dim ON job_postings_fact.job_id = skills_job_dim.job_id
INNER JOIN skills_dim ON skills_job_dim.skill_id = skills_dim.skill_id
WHERE
    job_title_short = 'Data Analyst'
    AND salary_year_avg IS NOT NULL
    AND job_work_from_home = True
GROUP BY skills
ORDER BY avg_salary DESC
LIMIT 25;
```

| Skill | Average Salary ($) |
|-------|------------------:|
| PySpark | 208,172 |
| Bitbucket | 189,155 |
| Couchbase | 160,515 |
| Watson | 160,515 |
| DataRobot | 155,486 |
| GitLab | 154,500 |
| Swift | 153,750 |
| Jupyter | 152,777 |
| Pandas | 151,821 |
| Elasticsearch | 145,000 |

**Three themes emerge from the top earners:**

**Big data and ML proficiency pay a premium.** PySpark, DataRobot, and Jupyter signal analysts who can operate closer to the engineering and modeling layer a rare combination that commands rare compensation.

**DevOps-adjacent skills are underrated.** GitLab, Kubernetes, and Airflow tools are typically associated with engineers. Analysts who understand deployment and pipeline management become significantly more valuable.

**Cloud fluency is becoming table stakes.** Elasticsearch, Databricks, and GCP appearing this high on the salary list reflects how deeply cloud-based analytics has embedded itself into the modern data stack.

---

### 5. The Optimal Skills Where Demand Meets Salary

This is the most strategically useful query of the project: skills that are both frequently requested *and* well-compensated. The intersection is where the highest ROI on learning time lives.

```sql
SELECT 
    skills_dim.skill_id,
    skills_dim.skills,
    COUNT(skills_job_dim.job_id) AS demand_count,
    ROUND(AVG(job_postings_fact.salary_year_avg), 0) AS avg_salary
FROM job_postings_fact
INNER JOIN skills_job_dim ON job_postings_fact.job_id = skills_job_dim.job_id
INNER JOIN skills_dim ON skills_job_dim.skill_id = skills_dim.skill_id
WHERE
    job_title_short = 'Data Analyst'
    AND salary_year_avg IS NOT NULL
    AND job_work_from_home = True
GROUP BY skills_dim.skill_id
HAVING COUNT(skills_job_dim.job_id) > 10
ORDER BY
    avg_salary DESC,
    demand_count DESC
LIMIT 25;
```

| Skill | Demand Count | Average Salary ($) |
|-------|-------------|------------------:|
| Go | 27 | 115,320 |
| Confluence | 11 | 114,210 |
| Hadoop | 22 | 113,193 |
| Snowflake | 37 | 112,948 |
| Azure | 34 | 111,225 |
| BigQuery | 13 | 109,654 |
| AWS | 32 | 108,317 |
| Java | 17 | 106,906 |
| SSIS | 12 | 106,683 |
| Jira | 20 | 104,918 |

**Reading between the lines:**

- **Python and R** lead in raw demand (236 and 148 postings respectively) with solid but not top-tier salaries (~$101K). They're essential, but they're not differentiators anymore.
- **Cloud platforms** (Snowflake, Azure, AWS, BigQuery) sit in the sweet spot: high enough demand to be widely relevant, high enough salary to justify specialization.
- **BI tools** (Tableau at 230 postings, Looker at 49) remain strong anchors, especially for analysts whose core output is business-facing dashboards and reports.
- **Niche database skills** (Oracle, SQL Server, NoSQL) round out the picture with $97K–$104K averages and consistent demand.

---

## What This Project Built

The technical output was a set of SQL queries. But the real output was a clearer framework for thinking about data careers.

Before this analysis, "learn SQL and Python" felt like generic advice. After it, the picture is more specific: master SQL first (it appears everywhere, at every salary level), then build cloud fluency (Snowflake, Azure, AWS are the skills with the best demand-to-salary ratio), and develop at least one strong visualization skill. The analysts pulling top-tier salaries are the ones who've extended into adjacent technical territory, pipelines, cloud infrastructure, ML tooling, without losing the analytical core.

The other thing this project reinforced: **data tells a cleaner story when you ask precise questions.** Each query was designed to isolate one variable at a time. Stacking the results revealed patterns that no single query could have shown alone.

---

## Key Takeaways

1. **The salary ceiling in data analytics is higher than most assume** - Remote roles reached $650K in 2023, driven by senior and director-level positions at major tech and finance companies.

2. **SQL is non-negotiable** - it leads in demand, appears consistently in high-paying roles, and delivers strong average compensation. There's no substitution.

3. **Niche skills carry a salary premium** - tools like PySpark, DataRobot, and Solidity aren't appearing in most job descriptions, but where they do, they pay significantly above market.

4. **Cloud is the highest-leverage investment** - Snowflake, Azure, and AWS offer a combination of broad availability and above-average compensation that makes them the clearest ROI play after the fundamentals.

5. **The analysts who earn the most look more like hybrid engineers** - they haven't abandoned analytics; they've extended into data infrastructure, cloud, and automation. That crossover skill set is where the real premium lives.

---

*Data sourced from [Luke Barousse's SQL Course](https://lukebarousse.com/sql) — job postings dataset covering 2023 titles, salaries, locations, and skills.*
