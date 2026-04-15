# Heartbeat Checklist

The heartbeat reads this file every tick and executes the checklist items. This is
an OBSERVATION-ONLY system. It flags findings via Discord and never takes external action.

Route findings to appropriate channels:
- Routine updates and project-context observations -> `#heartbeat`
- Commitment-related observations -> `#commitments`
- Cost digests and spend alerts -> `#cost-tracking`
- Actionable items requiring attention -> `#system-alerts`
- Items needing approval (should be rare from heartbeat) -> `#approvals`

Never take external action (no emails, no calendar invites, no pushes, no writes to
external systems). Heartbeat does NOT approve its own memory writes; any memory_write
it needs will prompt [Your Name] the same as any other caller.

90-day observation period applies. Review the `heartbeat_log` table weekly for the
first month to tune this file.

---

## Project Context Review (daily 9am)
- Check for project context older than 30 days with no updates
- Flag stale projects for review
- Route findings to `#heartbeat`

## Follow-up Tracking (every 4 hours)
- Surface follow-ups approaching their date based on project context
- Include relevant context entries sorted by date
- Route findings to `#heartbeat`

## Weekly Agent Digest (Monday 7am)
- Summarize what you worked on in the past week
- List active projects and their last update dates
- List work-related follow-ups coming due this week
- Include API spend from the past week
- Route to `#heartbeat`

## Commitment Digest (Monday 7:15am)
- Summarize open commitments grouped by age
- Flag overdue commitments (age > any explicit deadline in the signal)
- Summarize delegations awaiting follow-up
- Summarize decisions captured this week
- Route to `#commitments`

## Daily Cost Digest (daily 7pm)
- Report today's task count (local vs API)
- Report API token usage and estimated cost
- Report month-to-date spend and projection
- Route to `#cost-tracking`

## Cross-Project Relevance (daily 10am)
- Check if recent research overlaps with existing project context
- Flag connections worth noting
- Route to `#heartbeat`

## System Health (daily 8am)
- Verify last database backup exists and is recent
- Verify last workspace backup exists and is recent (v5.0 -- NEW)
- Check available disk space (alert if below 50GB)
- Route to `#system-alerts` if anything is wrong, `#heartbeat` if all green

## Router Health (every 15 minutes)
- Check Smart Router at localhost:3100/router/health
- Alert if unresponsive
- Route to `#system-alerts`
