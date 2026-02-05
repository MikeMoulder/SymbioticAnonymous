# Portfolio Submission & Review Flow - Complete Documentation

## Overview

This document explains the complete flow from when a user submits their portfolio until they receive a Discord role promotion.

```
┌─────────────┐    ┌─────────────┐    ┌─────────────┐    ┌─────────────┐    ┌─────────────┐
│   Website   │───▶│   Backend   │───▶│   Discord   │───▶│  Parliament │───▶│    Role     │
│   Submit    │    │   Review    │    │   Embed     │    │   Voting    │    │  Assigned   │
└─────────────┘    └─────────────┘    └─────────────┘    └─────────────┘    └─────────────┘
```

---

## Stage 1: User Submits Portfolio on Website

### What User Sees

When user clicks "Submit for Review" on `/portfolio` page:

1. **Optimistic UI Update** - Status immediately changes to "UNDER REVIEW"
2. **Success Message** - "Portfolio submitted for review!"
3. **Dashboard View** - Shows submission status with ⏳ icon

### Code Flow

**File:** `web/src/app/portfolio/page.tsx`

```typescript
const handleSubmitPortfolio = async () => {
  setSubmitting(true);
  
  // Optimistic UI update
  setDashboard({
    ...dashboard,
    portfolio: { ...dashboard.portfolio, status: 'submitted' }
  });
  
  // POST to API
  const res = await fetch('/api/portfolio/submit', { method: 'POST' });
  const data = await res.json();
  
  if (!data.success) {
    // Revert on failure
    fetchDashboard();
  }
};
```

### API Call Chain

```
Frontend                     Next.js API Route                    FastAPI Backend
───────────────────────────────────────────────────────────────────────────────────
POST /api/portfolio/submit → /api/portfolio/submit/route.ts → POST /api/portfolio/submit
                             (adds discord_id from session)      (updates DB status)
```

### Backend Processing

**File:** `backend/src/api/routers/portfolio.py`

```python
@router.post("/submit")
async def submit_portfolio(request: PortfolioSubmitRequest, session_maker):
    # 1. Find portfolio in DB
    portfolio = await session.execute(
        select(Portfolio).where(Portfolio.user_id == int(request.discord_id))
    )
    
    # 2. Validate status transitions
    if portfolio.status == "submitted":
        return {"success": False, "message": "Already submitted"}
    
    # 3. Check cooldown for rejected portfolios (7 days)
    if portfolio.status == "rejected" and portfolio.rejected_at:
        cooldown_end = portfolio.rejected_at + timedelta(days=7)
        if datetime.utcnow() < cooldown_end:
            return {"success": False, "message": "Cooldown active"}
    
    # 4. Update status
    portfolio.status = "submitted"
    portfolio.submitted_at = datetime.utcnow()
    await session.commit()
    
    return {"success": True, "status": "submitted"}
```

### Database Changes

```sql
UPDATE portfolios 
SET status = 'submitted', 
    submitted_at = NOW() 
WHERE user_id = {discord_id};
```

---

## Stage 2: Guild Leader Reviews on Website

### What Guild Leader Sees

On `/portfolios` page:

1. **Portfolio List** - Grid of all portfolios with status badges
2. **Filter Buttons** - All / Submitted / Draft
3. **Portfolio Modal** - Click to view full details
4. **Review Buttons** - "Approve" and "Reject" (only for Leads)

### Permission Check

**File:** `web/src/app/api/portfolio/review/route.ts`

```typescript
// Check if user has Lead role in Discord
async function checkIsLead(accessToken: string): Promise<boolean> {
  const memberResponse = await fetch(
    `https://discord.com/api/v10/users/@me/guilds/${DISCORD_GUILD_ID}/member`,
    { headers: { Authorization: `Bearer ${accessToken}` } }
  );
  
  const memberData = await memberResponse.json();
  const userRoles: string[] = memberData.roles || [];
  
  // LEAD_ROLE_IDS from environment
  return LEAD_ROLE_IDS.some(leadRoleId => userRoles.includes(leadRoleId));
}
```

### Approve Button Click

**File:** `web/src/app/portfolios/page.tsx`

```typescript
const handleReview = async (action: 'approve' | 'reject') => {
  setReviewing(true);
  
  const res = await fetch('/api/portfolio/review', {
    method: 'POST',
    body: JSON.stringify({
      portfolioUserId: selectedPortfolio.user_id,
      action,  // 'approve' or 'reject'
    }),
  });
  
  const data = await res.json();
  
  if (data.success) {
    setReviewMessage({
      type: 'success',
      text: action === 'approve' 
        ? 'Portfolio sent to Parliament for voting!' 
        : 'Portfolio rejected successfully',
    });
  }
};
```

### API Call Chain

```
Frontend                     Next.js API Route                    FastAPI Backend
───────────────────────────────────────────────────────────────────────────────────
POST /api/portfolio/review → /api/portfolio/review/route.ts → POST /api/portfolio/review
Body: {                      1. Checks Lead role via Discord      1. Updates status
  portfolioUserId,           2. Adds reviewer_id, reviewer_name   2. Creates embed
  action: 'approve'          3. Forwards to backend               3. Sends to Discord
}
```

---

## Stage 3: Backend Sends to Discord

### Backend Review Handler

**File:** `backend/src/api/routers/portfolio.py`

```python
@router.post("/review")
async def review_portfolio(request: PortfolioReviewRequest, session_maker, messages_repo):
    # 1. Get portfolio from DB
    portfolio = await session.execute(
        select(Portfolio).where(Portfolio.user_id == int(request.portfolio_user_id))
    )
    
    # 2. Validate status
    if portfolio.status != "submitted":
        return {"success": False, "message": "Not in submitted status"}
    
    # 3. Get user stats from messages.db
    stats = messages_repo.get_user_stats(int(request.portfolio_user_id))
    username = stats.get("username")
    message_count = stats.get("message_count", 0)
    days_active = stats.get("days_active", 0)
    tweets = messages_repo.get_user_tweets(int(request.portfolio_user_id))
    tweet_count = len(tweets)
    
    # 4. Get portfolio data
    portfolio_data = portfolio.ai_report or {}
    guild = portfolio_data.get("guild", "Unknown")
    twitter = portfolio_data.get("twitter_username", "")
    top_content = portfolio_data.get("top_content", "")
    unique_qualities = portfolio_data.get("unique_qualities", "")
    why_promotion = portfolio_data.get("why_promotion", "")
    current_role = portfolio.current_role or "Droplet"
    
    if request.action == "approve":
        # 5. Update status to pending_vote
        portfolio.status = "pending_vote"
        await session.commit()
        
        # 6. Create Discord embed
        embed = {
            "title": f"🗳️ Portfolio Vote: {username}",
            "description": f"**{request.reviewer_name}** approved for parliament voting.",
            "color": 0x00D4AA,
            "fields": [
                {"name": "👤 User", "value": f"<@{request.portfolio_user_id}>", "inline": True},
                {"name": "🎭 Guild", "value": guild, "inline": True},
                {"name": "🎖️ Current Role", "value": current_role, "inline": True},
                {"name": "💬 Messages", "value": str(message_count), "inline": True},
                {"name": "📅 Days Active", "value": str(days_active), "inline": True},
                {"name": "🐦 Tweets", "value": str(tweet_count), "inline": True},
                {"name": "🏆 Top Content", "value": top_content[:200], "inline": False},
                {"name": "✨ What Makes Them Unique", "value": unique_qualities[:200], "inline": False},
                {"name": "🎯 Why They Deserve Promotion", "value": why_promotion[:200], "inline": False},
            ],
            "footer": {"text": "Click to vote • Each member can only vote once"},
            "timestamp": datetime.utcnow().isoformat()
        }
        
        # 7. Create voting buttons
        components = [
            {
                "type": 1,  # Action Row
                "components": [
                    {
                        "type": 2,  # Button
                        "style": 3,  # Green
                        "label": "Approve",
                        "emoji": {"name": "✅"},
                        "custom_id": f"vote_approve_{request.portfolio_user_id}"
                    },
                    {
                        "type": 2,  # Button
                        "style": 4,  # Red
                        "label": "Reject",
                        "emoji": {"name": "❌"},
                        "custom_id": f"vote_reject_{request.portfolio_user_id}"
                    }
                ]
            }
        ]
        
        # 8. Send to Discord
        message_id = await send_discord_message_with_buttons(
            NOMINATION_CHANNEL_ID, 
            embed=embed, 
            components=components
        )
        
        # 9. Store message ID for later deletion
        portfolio.discord_message_id = int(message_id)
        await session.commit()
```

### Discord API Call

**File:** `backend/src/api/routers/portfolio.py`

```python
async def send_discord_message_with_buttons(channel_id: str, embed: dict, components: list):
    async with httpx.AsyncClient() as client:
        response = await client.post(
            f"https://discord.com/api/v10/channels/{channel_id}/messages",
            headers={
                "Authorization": f"Bot {DISCORD_BOT_TOKEN}",
                "Content-Type": "application/json"
            },
            json={
                "embeds": [embed],
                "components": components
            }
        )
        
        if response.status_code == 200:
            return response.json().get("id")  # Message ID
```

### Embed Structure in Discord

```
┌─────────────────────────────────────────────────────────────────┐
│ 🗳️ Portfolio Vote: Username                                    │
├─────────────────────────────────────────────────────────────────┤
│ **GuildLeadName** has approved this portfolio for voting.      │
├─────────────────────────────────────────────────────────────────┤
│ 👤 User          │ 🎭 Guild      │ 🎖️ Current Role             │
│ @Username        │ helpers       │ Droplet                     │
├─────────────────────────────────────────────────────────────────┤
│ 💬 Messages      │ 📅 Days Active │ 🐦 Tweets                  │
│ 1,234            │ 45             │ 12                         │
├─────────────────────────────────────────────────────────────────┤
│ 🏆 Top Content Highlights                                       │
│ https://x.com/user/status/123...                                │
├─────────────────────────────────────────────────────────────────┤
│ ✨ What Makes Them Unique                                       │
│ Active community helper, always welcoming newcomers...          │
├─────────────────────────────────────────────────────────────────┤
│ 🎯 Why They Deserve Promotion                                   │
│ Consistent contributions over 3 months...                       │
├─────────────────────────────────────────────────────────────────┤
│ Click a button to vote • Each member can only vote once         │
├─────────────────────────────────────────────────────────────────┤
│  [✅ Approve]  [❌ Reject]                                       │
└─────────────────────────────────────────────────────────────────┘
```

---

## Stage 4: Parliament Voting in Discord

### Vote Button Handler

**File:** `src/bot/commands/parliament.py`

```python
class ParliamentVoteView(discord.ui.View):
    def __init__(self, cog: 'ParliamentCog'):
        super().__init__(timeout=None)  # Persistent view
        self.cog = cog
    
    @discord.ui.button(label="✅ Approve", style=discord.ButtonStyle.green, custom_id="parliament_vote_yes")
    async def vote_yes(self, interaction: discord.Interaction, button: discord.ui.Button):
        await self._handle_vote(interaction, "yes")
    
    @discord.ui.button(label="❌ Reject", style=discord.ButtonStyle.red, custom_id="parliament_vote_no")
    async def vote_no(self, interaction: discord.Interaction, button: discord.ui.Button):
        await self._handle_vote(interaction, "no")
```

### Vote Processing

```python
async def _handle_vote(self, interaction: discord.Interaction, vote_type: str):
    embed = interaction.message.embeds[0]
    
    # 1. Extract user ID from embed
    for field in embed.fields:
        if field.name == "👤 Candidate":
            portfolio_user_id = field.value.split("<@")[1].split(">")[0]
    
    # 2. Get current vote count from embed
    for field in embed.fields:
        if field.name in ["📊 Current Vote", "📊 Final Vote"]:
            parts = field.value.split("/")
            votes_yes = int(parts[0].replace("✅", "").strip())
            votes_no = int(parts[1].replace("❌", "").strip())
    
    # 3. Increment vote
    if vote_type == "yes":
        votes_yes += 1
    else:
        votes_no += 1
    
    # 4. Update embed
    embed.set_field_at(vote_field_index, 
        name="📊 Current Vote",
        value=f"✅ {votes_yes} / ❌ {votes_no}"
    )
    await interaction.message.edit(embed=embed)
    
    # 5. Notify voter
    await interaction.response.send_message(
        f"{'✅' if vote_type == 'yes' else '❌'} Your vote has been recorded!",
        ephemeral=True
    )
    
    # 6. Check if vote threshold reached
    total_votes = votes_yes + votes_no
    if total_votes >= 5:
        approval_rate = votes_yes / total_votes
        
        if approval_rate >= 0.6:  # 60% approval
            await self.cog._finalize_vote(portfolio_user_id, approved=True, ...)
        elif (1 - approval_rate) > 0.4:  # Majority rejected
            await self.cog._finalize_vote(portfolio_user_id, approved=False, ...)
```

### Vote Thresholds

| Condition | Result |
|-----------|--------|
| 5+ votes AND 60%+ approve | ✅ **APPROVED** |
| 5+ votes AND 40%+ reject | ❌ **REJECTED** |
| < 5 votes | ⏳ Voting continues |

---

## Stage 5: Vote Finalization & Role Assignment

### Finalize Vote Handler

**File:** `src/bot/commands/parliament.py`

```python
async def _finalize_vote(self, portfolio_user_id, approved, votes_yes, votes_no, message):
    # 1. Update portfolio status in backend
    async with aiohttp.ClientSession() as session:
        async with session.post(
            f"{self.api_base_url}/api/portfolio/finalize",
            json={
                "portfolio_user_id": portfolio_user_id,
                "approved": approved,
                "votes_yes": votes_yes,
                "votes_no": votes_no
            }
        ) as resp:
            data = await resp.json()
            new_role = data.get("new_role")
    
    # 2. Update embed appearance
    embed = message.embeds[0]
    
    if approved:
        embed.color = 0x00ff00  # Green
        embed.title = "✅ Portfolio Approved - Promoted!"
        
        # 3. Assign Discord role
        if new_role and new_role in ROLE_IDS:
            guild = self.bot.get_guild(DISCORD_GUILD_ID)
            member = await guild.fetch_member(int(portfolio_user_id))
            role = guild.get_role(ROLE_IDS[new_role])
            
            await member.add_roles(role, reason=f"Parliament approved ({votes_yes}-{votes_no})")
            
            embed.add_field(
                name="🎖️ Promotion",
                value=f"**{new_role}** role assigned!",
                inline=True
            )
    else:
        embed.color = 0xff0000  # Red
        embed.title = "❌ Portfolio Rejected"
        embed.add_field(
            name="⏰ Cooldown",
            value="User can resubmit in 7 days",
            inline=True
        )
    
    # 4. Update vote count to "Final"
    embed.set_field_at(vote_field_index,
        name="📊 Final Vote",
        value=f"✅ {votes_yes} / ❌ {votes_no}"
    )
    
    # 5. Remove voting buttons
    await message.edit(embed=embed, view=None)
    
    # 6. Notify user via DM
    user = await self.bot.fetch_user(int(portfolio_user_id))
    
    if approved:
        await user.send(
            f"🎉 **Congratulations!** Your portfolio has been approved!\n\n"
            f"Final Vote: ✅ {votes_yes} / ❌ {votes_no}\n\n"
            f"🎖️ **You have been promoted to {new_role}!**"
        )
    else:
        await user.send(
            f"❌ Your portfolio was not approved.\n\n"
            f"Final Vote: ✅ {votes_yes} / ❌ {votes_no}\n\n"
            f"⏰ You may resubmit after **7 days**."
        )
```

### Role IDs

```python
ROLE_IDS = {
    "Droplet": 1445308513663324243,
    "Current": 1444006094283477085,
    "Founding_Droplet": 1069142027784171623,
    "Tide": 1069140617239744572,
    "Wave": 1068785740068159590,
    "Tsunami": 957362763267702815,
    "Allinliquid": 1377229341418852402,
}
```

---

## Complete Flow Diagram

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                              PORTFOLIO SUBMISSION FLOW                           │
└─────────────────────────────────────────────────────────────────────────────────┘

USER                     WEBSITE                   BACKEND                 DISCORD
────                     ───────                   ───────                 ───────
  │                         │                         │                       │
  │ Click "Submit"          │                         │                       │
  │────────────────────────▶│                         │                       │
  │                         │ POST /api/portfolio/    │                       │
  │                         │      submit             │                       │
  │                         │────────────────────────▶│                       │
  │                         │                         │ status='submitted'    │
  │                         │                         │ submitted_at=NOW()    │
  │                         │◀────────────────────────│                       │
  │◀────────────────────────│ "Under Review" UI      │                       │
  │                         │                         │                       │
  ╔═════════════════════════╪═════════════════════════╪═══════════════════════╪════╗
  ║ GUILD LEADER REVIEW     │                         │                       │    ║
  ╚═════════════════════════╪═════════════════════════╪═══════════════════════╪════╝
  │                         │                         │                       │
LEAD                        │ Click "Approve"         │                       │
  │────────────────────────▶│                         │                       │
  │                         │ POST /api/portfolio/    │                       │
  │                         │      review             │                       │
  │                         │────────────────────────▶│                       │
  │                         │                         │ status='pending_vote' │
  │                         │                         │                       │
  │                         │                         │ POST Discord API      │
  │                         │                         │ Send embed+buttons    │
  │                         │                         │──────────────────────▶│
  │                         │                         │                       │ 📋 Embed
  │                         │                         │                       │ [✅][❌]
  │                         │                         │◀──────────────────────│ msg_id
  │                         │                         │ Store discord_msg_id  │
  │                         │◀────────────────────────│                       │
  │◀────────────────────────│ "Sent to Parliament"   │                       │
  │                         │                         │                       │
  ╔═════════════════════════╪═════════════════════════╪═══════════════════════╪════╗
  ║ PARLIAMENT VOTING       │                         │                       │    ║
  ╚═════════════════════════╪═════════════════════════╪═══════════════════════╪════╝
  │                         │                         │                       │
MEMBER                      │                         │                       │ Click ✅
  │                         │                         │                       │────────▶
  │                         │                         │                       │ Increment
  │                         │                         │                       │ votes_yes
  │                         │                         │                       │ Update embed
  │                         │                         │                       │
  │                         │                         │           (5+ votes, 60%+ approve)
  │                         │                         │                       │
  │                         │                         │◀──────────────────────│
  │                         │                         │ POST /api/portfolio/  │
  │                         │                         │      finalize         │
  │                         │                         │                       │
  │                         │                         │ status='approved'     │
  │                         │                         │ Archive to history    │
  │                         │                         │                       │
  │                         │                         │──────────────────────▶│
  │                         │                         │                       │ Assign role
  │                         │                         │                       │ Update embed
  │                         │                         │                       │ Remove buttons
  │                         │                         │                       │
USER                        │                         │                       │ Send DM
  │◀────────────────────────────────────────────────────────────────────────────│
  │ "🎉 Congratulations! Promoted to Current!"        │                       │
```

---

## Database Status Transitions

```
draft ──────────▶ submitted ──────────▶ pending_vote ──────────▶ approved/rejected
        User submits    Lead approves        Parliament votes
                        on website           in Discord
```

| Status | Description | Who Changes | Where |
|--------|-------------|-------------|-------|
| `draft` | User editing | User | Website |
| `submitted` | Awaiting lead review | User | Website |
| `pending_vote` | In parliament voting | Lead | Website → Discord |
| `approved` | Passed vote | Bot | Discord |
| `rejected` | Failed vote | Bot | Discord |
| `promoted` | Role assigned | Bot | Discord |

---

## Key Files Summary

| File | Purpose |
|------|---------|
| `web/src/app/portfolio/page.tsx` | User portfolio page (submit button) |
| `web/src/app/portfolios/page.tsx` | Lead review page (approve/reject) |
| `web/src/app/api/portfolio/submit/route.ts` | Submit API proxy |
| `web/src/app/api/portfolio/review/route.ts` | Review API proxy (checks Lead role) |
| `backend/src/api/routers/portfolio.py` | Backend portfolio endpoints |
| `src/bot/commands/parliament.py` | Discord voting buttons & finalization |

---

## Environment Variables Required

```env
# Discord
DISCORD_TOKEN=bot_token
DISCORD_GUILD_ID=957362763267702814
NOMINATION_CHANNEL_ID=1439139346241556480

# Lead Role IDs (comma-separated)
LEAD_ROLE_IDS=123,456,789

# Role IDs for promotion
Droplet=1445308513663324243
Current=1444006094283477085
Tide=1069140617239744572
Wave=1068785740068159590
Tsunami=957362763267702815
Allinliquid=1377229341418852402

# Backend
BACKEND_API_URL=http://localhost:8000
```

---

## Notifications to User

| Event | Message | Via |
|-------|---------|-----|
| Submit | "Portfolio submitted for review!" | Website toast |
| Lead Approve | (no notification) | — |
| Lead Reject | "Portfolio rejected" | Website + DM |
| Vote Pass | "🎉 Promoted to [Role]!" | Discord DM |
| Vote Fail | "❌ Not approved. Resubmit in 7 days." | Discord DM |
