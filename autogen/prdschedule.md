# Production Scheduling - Using Autogen to Generate a Production Schedule

## Introduction

- A system to do production planning and scheduling.
- Given the existing capacity and demand, generate a production schedule.
- Provide total units produced, total capacity and resource used and available.
- Using Autogen Studio as UI to generate the production schedule.
- Only for knowledge.
- Not for production use.

## Steps

- First install the Autogen Studio.
- also install autogen package.

```
autogen-agentchat
autogenstudio
autogen-ext
autogen-ext[openai]
autogen-ext[azure]
openai
uv
```

- I am using Azure Open AI service to run the model.
- Using gpt 4.1 model.
- Create a new Team in Autogen Studio.
- Plant Manager system prompt:

```
You are a Plant Manager helpful assistant. Solve tasks carefully. Make sure the given problem get the plan, schedule and production completed by time. Here is the context of capacity of the plant floor: Plant can produce 1000000 units of water bottles a month. There are about 20 workers working full time in 2 shifts, each shift is 8 hours each. As of right now there is 10 workers working and about 200000 bottles are produced a month.
When done, say TERMINATE.
```

- System prompt for Production Planner:

```
You are a Production Planner / Scheduler AI assistant. Solve tasks carefully. Provide details on the schedule and planner planned to execute for the above task. Consider 3 shifts based on availability of resource and line capacity. Try to keep the production within date of delivery. Here is the context of capacity of the plant floor: Plant can produce 1000000 units of water bottles a month. There are about 20 workers working full time in 2 shifts, each shift is 8 hours each. As of right now there is 10 workers working and about 200000 bottles are produced a month. When done, say TERMINATE.
```

![info](https://github.com/balakreshnan/Samples2025/blob/main/autogen/images/prdplanning-5.jpg 'RagChat')

- here is the output of the team creation:

```
i would like to produce 500,000 water bottles
```

- Here is the output of production schedule generation:
- Plant manager output

```
PlantManager

Here’s a step-by-step plan to produce 500,000 water bottles:

Assessment of Current Capacity:
Current workforce: 10 workers producing 200,000 bottles/month.
Plant maximum capacity: 1,000,000 bottles/month with a full workforce (20 workers).
You require 500,000 bottles, which is 2.5 times your current monthly output.
Calculation of Resources Needed:
Since 10 workers = 200,000 bottles, the current productivity per worker/month is 20,000 bottles.
For 500,000 bottles: 500,000 ÷ 20,000 = 25 worker-months needed.
Options for Scheduling and Staffing: Option A: Increase workforce back up to 20 workers, resume 2 full shifts, and aim for higher output.
20 workers × 20,000 bottles/worker = 400,000 bottles/month.
To make 500,000 bottles, you’ll need 1.25 months (about 5 weeks) at full staffing.
Option B: Keep 10 workers and increase production time:

10 workers can produce 200,000 bottles/month, so it’ll take 2.5 months to hit 500,000 bottles.
Recommendations:
Hire 10 more workers to reach full capacity if you need the order quickly.
If timing is flexible, stretch production over 2.5 months with current staff.
Schedule: If you add more workers:
Ramp up hiring and training immediately.
Set a shift plan to maximize use of both 8-hour shifts.
Plan to complete production in just over 1 month.
If you do not hire:

Maintain current production pace and deliver in 2.5 months.
Would you like to optimize for speed by hiring or maintain your current pace? Let me know, and I’ll prepare the specific production schedule and daily targets.

Tokens: 580
```

- Production planner output

```
ProductionPlannerScheduler

Here’s the detailed production scheduling plan for your order of 500,000 water bottles, considering your plant’s resources and capacity.

Current State Overview:

Current workers: 10
Current production: 200,000 bottles/month (10 workers, 2 shifts, 8 hours each)
Maximum possible with full 20-worker staffing: 1,000,000 bottles/month
Target Output:

Required: 500,000 bottles
Production Options:

A. Using Current Staff (10 workers, 2 shifts):

Production rate: 200,000 bottles/month
Time required: 500,000 ÷ 200,000 = 2.5 months
B. Double the Workforce (20 workers, 2 shifts):

Production rate: 400,000 bottles/month (assuming linear increase with workers; historically max plant output is 1,000,000/month, but per your real operation, 20 workers = 400k/month)
Time required: 500,000 ÷ 400,000 = 1.25 months (about 5 weeks)
C. Add a Third Shift (assuming you can staff it): If resources allow, running 3 shifts (requires total 15 workers minimum, based on current productivity):

Estimated output: 300,000 bottles/month (with 15 workers in 3 shifts)
Time required: 500,000 ÷ 300,000 = 1.67 months
Recommended Plan: If delivery is urgent, hire back up to 20 workers and/or introduce a third shift if feasible.

PRODUCTION SCHEDULE (assuming you stick to Option B: 20 workers with 2 shifts): Week 1: Ramp up workforce, training, swing to 20 workers. Production Target: 100,000 bottles/week (assuming linear output per week, 400,000/month ÷ 4) Weeks 2–5: Full production, 100,000 bottles/week. End of Week 5: Complete 500,000 bottles.

Shift Assignment Example:

Shift 1: 10 workers, 8am–4pm (50,000 bottles/week)
Shift 2: 10 workers, 4pm–12am (50,000 bottles/week)
DAILY PRODUCTION TARGET: 100,000 ÷ 5 ≈ 20,000 bottles per shift per day

Monitor daily output, adjust labor if a shortfall occurs.
Maintain inventory tracking to ensure all finished goods meet quality and timeline targets.
If only current workforce (Option A):

Same load, but spread over 2.5 months, or ~8 weeks, at 25,000 bottles/week.
Please confirm your staffing and shift preference, and I’ll finalize the daily dashboard and handover plan.

TERMINATE



Tokens: 1183
```

- Here are the screenshots of the Autogen Studio UI:

![info](https://github.com/balakreshnan/Samples2025/blob/main/autogen/images/prdplanning-1.jpg 'RagChat')

![info](https://github.com/balakreshnan/Samples2025/blob/main/autogen/images/prdplanning-2.jpg 'RagChat')

![info](https://github.com/balakreshnan/Samples2025/blob/main/autogen/images/prdplanning-3.jpg 'RagChat')

![info](https://github.com/balakreshnan/Samples2025/blob/main/autogen/images/prdplanning-4.jpg 'RagChat')

- Done.

## Conclusion

- This is a simple example of how to use Autogen to generate a production schedule.
- Idea is here to show how to use Autogen to generate a production schedule.
- You can use this as a starting point to build more complex production scheduling systems.