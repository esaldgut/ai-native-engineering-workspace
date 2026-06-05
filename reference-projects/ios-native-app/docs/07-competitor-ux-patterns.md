# 07 — Competitor UX/UI Design Patterns for MyApp

Research compiled 2026-03-27 from design case studies, engineering blogs, UX pattern
libraries, and official documentation. Only verified patterns with sources.

---

## 1. Profile Page Patterns

### 1.1 Instagram Profile

**Source:** [Eleken Profile Page Design](https://www.eleken.co/blog-posts/profile-page-design), [Instagram Help Center](https://help.instagram.com), [Kapwing Grid Guide](https://www.kapwing.com/resources/instagrams-new-grid-layout-size-and-dimensions-2025/)

**Header layout:**
- Compact horizontal header: avatar (left) | stats bar (right)
- Avatar: 77pt circular, tap opens story ring if active
- Name: bold, left-aligned below avatar
- Pronouns: light grey, inline after name (up to 4)
- Bio: up to 150 characters, supports line breaks and emojis
- Website: single tappable link below bio (or Linktree-style multi-link)
- Category label: shown for Creator/Business accounts (e.g., "Digital Creator", "Restaurant")

**Stats bar (right of avatar):**
- Three columns: Posts | Followers | Following
- Number bold, label small grey below
- Each stat is tappable: Posts scrolls to grid; Followers/Following opens respective list
- Counts abbreviate at 10K+ (e.g., "1.2M")

**Action buttons (below header):**
- Own profile: `[Edit Profile]` `[Share Profile]` `[Discover People]`
- Other user (public): `[Follow]` `[Message]` `[▼]` (more options)
- Other user (following): `[Following ▼]` `[Message]` `[▼]`
- Buttons use rounded rectangle shape, full-width, horizontally stacked

**Story Highlights:**
- Horizontal scroll of circular thumbnails below action buttons
- Last item is "New" with `+` icon for adding highlights
- Tap opens full-screen story viewer

**Tab switching (below highlights):**
- Icon-only tabs: Grid (3x3) | Reels (play icon) | Tagged (person icon)
- Active tab has underline indicator
- Tabs become sticky on scroll (pin to top under navigation bar)
- Grid: 3-column, 3:4 aspect ratio thumbnails (updated January 2025, previously 1:1)
- Reels: 3-column, 9:16 thumbnails with play count overlay
- Tagged: same grid, showing posts where user is tagged

**Verified badge:**
- Blue checkmark inline after username
- Same visual for traditional verification and Meta Verified ($14.99/mo subscription)
- Appears in profile header, search results, comments, DMs

**Creator vs Business account distinctions:**
- Creator: category label, creator-specific analytics, music library access
- Business: category label + contact buttons (Call, Email, Directions), shopping features
- Both show "Professional dashboard" link on own profile

**Source:** [Meta Verified](https://www.meta.com/meta-verified/), [Instagram Help Center - Verified Badges](https://help.instagram.com/854227311295302)

### 1.2 TikTok Profile

**Source:** [Tamidy - TikTok UI/UX](https://www.tamidy.com/blog/the-ui-ux-of-tiktok-first-impressions), [Iterators - TikTok UI](https://www.iteratorshq.com/blog/5-tiktok-ui-choices-that-made-the-app-successful/)

**Header layout:**
- Centered layout (unlike Instagram's left-aligned)
- Avatar: centered, large circular image
- Username: centered below avatar, preceded by `@`
- Bio: centered, up to 160 characters (increased from 80 in 2025)
- Website link below bio

**Stats bar (horizontal, centered):**
- Three columns: Following | Followers | Likes
- Likes = total likes across all videos (credibility metric)
- Tappable to navigate to respective lists

**Action buttons:**
- Own profile: `[Edit Profile]` `[Share Profile]` `[Add Friends]`
- Other user: `[Follow]` `[Message]` `[▼]` (more)
- Follow button is prominent red/pink

**Tab switching:**
- Videos (grid icon) | Favorites (bookmark) | Liked (heart)
- Favorites subsections: Videos, Hashtags, Sounds, Effects, Products, Comments, Questions
- Videos tab: 3-column grid, 9:16 thumbnails with view count overlay
- Tabs become sticky on scroll

**Settings access:**
- Hamburger menu (top-right) opens bottom sheet with "Creator tools" and "Settings and privacy"

### 1.3 LinkedIn Profile

**Source:** [LinkedIn UI Case Study](https://medium.com/@vlad.derdeicea/linkedin-ui-case-study-ad356d131837), [Eleken Profile Page Design](https://www.eleken.co/blog-posts/profile-page-design), [LinkedIn Help Center](https://www.linkedin.com/help/linkedin/answer/a702683)

**Header layout:**
- Banner image: full-width, ~120pt tall, customizable
- Avatar: rounded-square (not circular), overlapping banner bottom edge, left-aligned
- Name + headline: below avatar, left-aligned
- Headline: professional tagline (e.g., "Senior iOS Developer at Company")
- Location + industry below headline
- Mutual connections count: "12 mutual connections" with overlapping avatar thumbnails

**Action buttons:**
- Primary: `[Connect]` (solid blue) or `[Follow]` (outline)
- Secondary: `[Message]` `[More ▼]`
- When connected: `[Message]` becomes primary, `[More ▼]` includes Unfollow/Remove/Block
- Creator Mode: replaces Connect with Follow as primary action

**Content sections (vertical scroll, not tabs):**
- About (bio)
- Experience
- Education
- Skills & Endorsements
- Recommendations
- Activity (recent posts)
- Each section is a collapsible card with "Show all" expansion

---

## 2. Social Graph Patterns

### 2.1 Follow Button States

**Source:** [UI Patterns - Follow](https://ui-patterns.com/patterns/follow), [LinkedIn Help](https://www.linkedin.com/help/linkedin/answer/a702683), [Instagram Help Center](https://help.instagram.com)

**Instagram follow states:**
1. `[Follow]` — blue/primary filled button (not following)
2. `[Following]` — grey/outline button (currently following)
3. `[Requested]` — grey/outline button (pending approval on private account)
4. Tap "Following" → confirmation sheet: "Unfollow @username?" with Unfollow/Cancel
5. Private account: Follow → Requested → (accepted) → Following

**TikTok follow states:**
1. `[Follow]` — red/pink filled button
2. `[Friends]` — grey outline (mutual follow)
3. Follow back: shows `[Follow back]` when they follow you but you don't follow them

**LinkedIn connection states:**
1. `[Connect]` — blue filled (no connection)
2. `[Pending]` — grey outline (request sent, awaiting acceptance)
3. `[Message]` — replaces Connect once connected (primary action shifts)
4. `[Follow]` — outline (available on Creator Mode profiles, or when Connect is hidden)
5. `[Following]` — grey (currently following, tap to unfollow)

### 2.2 Connection Request Flow (LinkedIn)

**Source:** [LinkedIn Help - Follow and Connect](https://www.linkedin.com/help/linkedin/answer/a702683), [Mobbin - LinkedIn Connect Flow](https://mobbin.com/explore/flows/6bf07322-45de-4631-8d91-55c20fe1a9b1)

1. Tap `[Connect]` on profile
2. Optional: Add a note (200 char limit) — modal sheet
3. Request sent → button changes to `[Pending]`
4. Recipient sees notification + "Accept / Ignore" in My Network tab
5. Accept → both become connections, mutual messaging enabled
6. Sending a connect request automatically follows the person
7. If request is ignored/withdrawn, user reverts to Follow state

### 2.3 Mutual Connections/Followers Display

**Instagram:**
- "Followed by [avatar] username and [N] others" shown on profile header
- Tappable to see full list of mutual followers
- Mutual followers list shows avatars + follow buttons

**LinkedIn:**
- "12 mutual connections" with 3 overlapping circular avatars
- Tappable to see full mutual connections list
- Shown prominently below name/headline

### 2.4 Suggested Connections ("People You May Know")

**Source:** [LinkedIn Help - Connections You May Know](https://www.linkedin.com/help/linkedin/answer/a553108)

**LinkedIn:**
- "My Network" tab: horizontal carousel of suggestion cards
- Card: avatar + name + headline + mutual connections count + `[Connect]` button
- Suggestion triggers: accepted a new connection → shows their connections
- Dismissible with `X` button per card
- Algorithm factors: mutual connections, company, school, industry, location

**Instagram:**
- "Suggested for you" section on Explore and after following someone
- Grid of profile cards: avatar + username + mutual followers + `[Follow]` button
- "Discover People" accessible from profile (person+ icon)
- Contact sync: suggests phone contacts who are on Instagram

### 2.5 Block/Report Flow

**Common pattern across Instagram, TikTok, LinkedIn:**
1. Tap `...` or `▼` menu on profile
2. Bottom sheet: Report | Block | Restrict | Mute (varies by platform)
3. Block: confirmation dialog → "Block @username? They won't be able to find your profile..."
4. Bidirectional: blocks all interaction, removes from followers, hides content
5. Report: multi-step flow → category selection → optional details → submit

### 2.6 Privacy Settings

**Instagram:**
- Public vs Private account toggle in Settings
- Private: Follow requires approval (Requested state), posts hidden from non-followers
- Close Friends: separate list for Stories-only sharing
- Restricted accounts: can see content but comments are hidden from others

**TikTok:**
- Public vs Private account toggle
- Private: followers require approval
- Per-video privacy: individual videos can be set private
- Download permissions: toggle to allow/disallow others downloading your videos

---

## 3. Marketplace / E-commerce Patterns

### 3.1 Product/Listing Card Design (Airbnb)

**Source:** [Airbnb Engineering - Server-Driven UI](https://medium.com/airbnb-engineering/a-deep-dive-into-airbnbs-server-driven-ui-system-842244c5f5), [It's Nice That - Airbnb Redesign](https://www.itsnicethat.com/articles/airbnb-app-redesign-140525), [Airbnb Summer 2025](https://medium.com/design-bootcamp/airbnb-summer-2025-update-heres-what-s-new-and-why-it-matters-0ced2338b921)

**Card anatomy (current Summer 2025 design):**
- Image carousel: horizontal swipeable (dots indicator below), rounded corners
- Heart icon: top-right corner of image for wishlist save (translucent background)
- Title: property name or host descriptor ("Treehouse in Bali")
- Location: city/region
- Rating: star icon + numeric rating (e.g., "4.92") + review count
- Price: "$XXX night" — bold price, "night" in normal weight
- Total price toggle: "Show total before taxes" option in search settings
- Superhost badge: shield icon + "Superhost" label
- Guest Favorite badge: replaces Superhost for top-rated listings
- Card style: full-bleed images, no card border — content-first

**Airbnb Summer 2025 visual changes:**
- 3D skeuomorphic icons (Pixar-inspired), vibrant and tactile
- Smooth animations with subtle lighting and soft curves
- Three content pillars: Homes, Experiences, Services
- New custom icon system with "universal language" design

**Server-Driven UI (Ghost Platform):**
- Sections are primitive building blocks containing display-ready data
- `SectionComponentType` controls rendering per context (same data, different visual)
- Unified GraphQL schema across web, iOS, Android
- Backend centralizes business logic; client renders declaratively

### 3.2 Product Card Design (Booking.com)

**Source:** [DesignRush - Booking.com](https://www.designrush.com/best-designs/apps/bookingcom), [Booking.com Partners](https://partner.booking.com/en-gb/help/growing-your-business/analytics-reports/search-results-ranking-and-visibility)

**Card anatomy:**
- Property photo: landscape thumbnail, left or top placement
- Property name + star rating (hotel stars, not review stars)
- Location: distance from city center
- Review score: circular badge with numeric rating (e.g., "8.7") + descriptor ("Excellent")
- Review count: "(2,345 reviews)"
- Price: "US$XXX" per night, with strikethrough for discounted original price
- Tax/fee note: "+ US$XX taxes and fees" in smaller text
- "Limited availability" urgency label (orange/red text)
- Booking count: "Booked X times in last 24 hours" (social proof)
- Free cancellation badge (green text when applicable)

**Category filter chips:**
- Horizontal scroll bar at top of results
- Categories: "All", "Popular", "Nearby", "Recommend", property types
- Active chip is filled/highlighted, inactive are outline

### 3.3 Gallery View Patterns

**Airbnb gallery:**
- Listing detail: hero image (large) + 4 smaller grid images
- Tap opens full-screen gallery with horizontal swipe
- Image counter: "3/24" overlay
- Share and Save buttons in gallery toolbar
- Pinch to zoom supported in full-screen
- Map preview as last gallery "image"

**Booking.com gallery:**
- Listing detail: horizontal image carousel at top
- Dot indicators for position
- Tap opens full-screen viewer
- Category tabs within gallery: "Room", "Bathroom", "View", "Facilities"

### 3.4 Booking/Reservation Flow (Airbnb)

**Source:** [Airbnb Booking UX Case Study](https://medium.com/design-bootcamp/case-study-airbnb-improvement-of-airbnb-booking-ux-process-beb51da06fd3), [Airbnb Help Center](https://www.airbnb.com/help/article/252)

**Step-by-step wizard:**
1. **Select dates** — Calendar picker (inline on listing page)
2. **Select guests** — Stepper controls (Adults, Children, Infants, Pets)
3. **Review & Pay** — Single confirmation page:
   - Trip details summary
   - Price breakdown (nightly rate x nights + cleaning fee + service fee + taxes)
   - Payment method selection
   - Message to host (optional)
   - Cancellation policy
   - `[Confirm and Pay]` primary CTA
4. **Confirmation** — Booking confirmed with trip details + next steps

**Instant Book vs Request:**
- Instant Book: one-step confirmation, no host approval needed
- Request to Book: host has 24h to accept/decline

**Group payment split (proposed UX improvement):**
- Pay Full Amount vs Split the Cost options
- Add additional guests by email
- Equal split or custom amounts per guest
- Real-time status tracking: Processing → Done

### 3.5 Date Picker Patterns

**Source:** [Storyly - Date Picker Examples](https://www.storyly.io/post/best-user-experience-datepicker-examples-for-mobile-and-web), [Mobiscroll - Booking Calendar](https://blog.mobiscroll.com/how-to-build-amazing-booking-apps-calendar-tips-and-considerations/)

**Airbnb date picker:**
- Inline calendar on listing page (not modal)
- Two-month view side by side (landscape) or scrollable vertical (portrait)
- Tap start date → tap end date → range highlighted
- Unavailable dates greyed out
- Price per night shown below each date (when available)
- Flexible dates toggle: "Exact dates" | "±1 day" | "±3 days" | "±7 days"
- Minimum stay enforced visually (dates before minimum grayed after check-in selection)

**Booking.com date picker:**
- Modal calendar overlay
- Single month view, vertically scrollable
- Check-in selection → auto-advances to check-out
- Maximum 30-day stay enforced
- Must start within 365 days
- Invalid dates automatically prevented
- Check-out cannot precede check-in (auto-adjusted)

**Best practices for travel apps:**
- Show price per night below each date for rate shopping
- Prevent invalid date selection rather than showing error after
- Highlight selected range with continuous background color
- Show day-of-week headers (S M T W T F S)
- Mark today's date distinctly
- Support swipe gestures for month navigation

### 3.6 Guest Count Selector

**Airbnb pattern:**
- Category steppers: Adults (13+), Children (2-12), Infants (under 2), Pets
- `-` and `+` buttons flanking a number
- Minimum: 1 adult, 0 for others
- Maximum enforced per listing
- Descriptive age ranges shown
- Service animals note: "Service animals are not pets"

### 3.7 Wishlist / Save for Later

**Source:** [Airbnb Wishlist](https://www.hardreset.info/devices/apps/apps-airbnb/create-wishlist/), [GoodUI - Airbnb A/B Test](https://goodui.org/leaks/airbnb-a-b-tests-and-detects-a-better-placement-for-saving-properties/)

**Airbnb:**
- Heart icon on listing card (top-right of image)
- Tap → bottom sheet: "Save to wishlist" with existing lists + "Create new list"
- Shared wishlists: collaborators can add notes, vote on listings
- A/B tested placement: heart on card vs. separate save button — card placement won

**Booking.com:**
- Heart/bookmark icon on property card
- "Saved" section in account navigation
- No collaborative wishlist feature

**Instagram Save (comparable pattern):**
- Bookmark icon below post
- Tap → saved to default collection
- Long-press → "Save to collection" bottom sheet
- Collections are private by default

### 3.8 Reviews and Ratings Display

**Airbnb:**
- Overall rating: star icon + numeric (4.92 scale)
- Category ratings: Cleanliness, Accuracy, Communication, Location, Check-in, Value
- Each category: horizontal bar chart or star display
- Reviews sorted by most recent, filterable
- Host response shown inline below relevant reviews
- Superhost badge: 4.8+ overall, 10+ stays, <1% cancellation, 90% response rate
- Guest Favorite badge: top 5% of listings based on ratings + reviews

**Booking.com:**
- Numeric score: 0-10 scale (e.g., "8.7 Excellent")
- Category scores: Staff, Facilities, Cleanliness, Comfort, Value, Location, WiFi
- Score displayed in colored circular badge (blue spectrum)
- Reviewer nationality flag shown
- Pros/Cons format: "Liked: ..." / "Disliked: ..."
- Filterable by guest type (Couples, Families, Solo, Business)

### 3.9 Map Integration

**Source:** [Airbnb Map Platform](https://adamshutsa.com/map-platform/), [Airbnb Engineering - Map Ranking](https://medium.com/airbnb-engineering/improving-search-ranking-for-maps-13b03f2c2cca)

**Airbnb:**
- Map view toggle on search results (top-right button)
- Price pins: oval badges with price ("$125") on map
- Two-tier pin system: high-booking-probability = regular oval; others = smaller dots
- Tap pin → mini listing card (image + title + rating + price)
- Drag map → auto-refreshes results for visible area
- Custom pin design: aligned with Design Language System
- Zoom levels control pin density and label visibility

**Booking.com:**
- Map view accessible from search results
- Pins with price labels
- Cluster pins at lower zoom levels
- Property card popup on pin tap
- "Search as I move the map" toggle

### 3.10 Filter and Sort Patterns

**Airbnb filters:**
- Modal full-screen filter view
- Sections: Price range (slider with histogram), Type of place (checkboxes), Rooms and beds (steppers), Amenities (checkbox grid), Property type (icon cards), Booking options (toggles)
- "Show X results" live counter on Apply button
- "Clear all" reset option
- Category chips at top of search: Icons, Beach, Countryside, Amazing pools, etc. (horizontal scroll)
- Icon-driven categories with illustration style

**Booking.com filters:**
- Side panel or full-screen modal on mobile
- Star rating filter (1-5 stars)
- Review score filter (3.0-5.0 range)
- Price range slider
- Amenity checkboxes with result counts
- Distance from center
- Sort options: Price (low-high), Rating, Distance, Popularity
- Filter chips bar at top with quick toggles

### 3.11 Search with Autocomplete

**Airbnb:**
- Search bar: "Where to?" placeholder
- Tap → full-screen search with sections:
  - Recent searches
  - Trending destinations
  - "I'm flexible" option (category-based: Beach, Mountains, etc.)
- Autocomplete: destinations, regions, experiences
- Each suggestion: location name + region + country + small thumbnail

**Booking.com:**
- Search bar: "Where are you going?" placeholder
- Autocomplete: hotels, cities, landmarks, airports, regions
- Each suggestion: icon (hotel/city/landmark) + name + type label + country
- Recent searches shown on focus

---

## 4. Onboarding Patterns

### 4.1 Instagram Onboarding

**Source:** [Appcues - Instagram Business Onboarding](https://goodux.appcues.com/blog/instagrams-business-onboarding), [Mobbin - Instagram Onboarding](https://mobbin.com/explore/flows/fc508ec1-6af0-46bd-9b01-578751475faa), [Page Flows - Instagram](https://pageflows.com/post/ios/onboarding/instagram/)

**Sign-up flow:**
1. Welcome screen: "Instagram" logo + `[Create new account]` + `[Log in]`
2. Social sign-in: `[Continue with Facebook]` prominent; Apple Sign-In via system sheet
3. Phone or email entry (toggle between phone/email)
4. Confirmation code (6-digit SMS or email verification)
5. Name entry (full name)
6. Password creation (minimum requirements shown inline)
7. Birthday entry (date wheel picker, must be 13+)
8. Username suggestion (auto-generated from name, editable)
9. Profile photo (optional, skip available): camera, library, or import from Facebook
10. Find contacts (optional): sync phone contacts
11. Follow suggestions: curated list of popular accounts by category

**Design principles:**
- No more than 30 words per step
- Minimal visual noise
- Every element serves a purpose
- Skip options on non-critical steps
- Social proof: "X of your friends are on Instagram"

**Business account onboarding (upgrade flow):**
- "Professional dashboard" entry point on profile
- Highlights key features with value before commitment
- "Not a business?" exit option prominently shown
- Maximum 30 words per step
- Social proof: "104 of the people you follow are already on a business profile"

### 4.2 TikTok Onboarding

**Source:** [Appcues - TikTok Onboarding](https://goodux.appcues.com/blog/tiktok-user-onboarding), [Tamidy - TikTok UI/UX](https://www.tamidy.com/blog/the-ui-ux-of-tiktok-first-impressions)

**Sign-up flow:**
1. No welcome screen — content plays immediately (For You feed visible before sign-up)
2. Sign-up prompted only when user tries to interact (like, comment, follow)
3. Sign-up modal: Phone/Email | Facebook | Apple | Google (Twitter hidden in dropdown)
4. Date of birth (required, must be 13+)
5. Phone/email entry + verification code
6. Password creation
7. Username auto-generated (editable later)

**Interest selection:**
- Full-screen grid of large category buttons (not checkboxes)
- Categories: Comedy, Dance, Food, Travel, Fashion, Sports, Gaming, etc.
- Multi-select, minimum selection encouraged
- Used to seed initial For You page algorithm

**Tutorial:**
- Animated demonstration of vertical swipe gesture
- "Swipe up for more videos" overlay on first video
- Mobile tooltip showing how to create first video (positioned at + button)
- Progressive: browse first → create later approach

**Permission requests (progressive):**
- Notification permission: asked when user first opens Inbox/notifications
- Camera: asked when user taps Create (+)
- Microphone: asked when starting video recording
- Contacts: asked when user taps "Find friends"
- Each permission at the "most appealing moment" — when user needs the feature

### 4.3 Airbnb Onboarding

**Source:** [Airbnb UX Case Study](https://medium.com/@vishal.peshne/ux-case-study-the-success-of-airbnbs-user-centered-approach-7557f3d769b9), [Airbnb Redesign 2025](https://www.itsnicethat.com/articles/airbnb-app-redesign-140525)

**Sign-up flow:**
1. Welcome: browsing enabled without account (delayed registration)
2. Sign-up prompted at booking, saving, or messaging
3. Methods: Email | `Continue with Apple` | `Continue with Google` | `Continue with Facebook` | Phone
4. Email flow: email → name → password → birthday → profile photo (optional)
5. Phone flow: phone number → SMS code → name → password
6. Government ID verification prompted later (before first booking)

**Progressive onboarding:**
- Full browsing without account (search, view listings, map)
- Account required only for: booking, messaging host, saving to wishlist
- Host onboarding is separate multi-step wizard
- "Trips" tab transforms into living itinerary post-booking

### 4.4 Social Sign-In Button Layout (Cross-Platform Standard)

**Source:** [Apple HIG - Sign in with Apple](https://developer.apple.com/design/human-interface-guidelines/sign-in-with-apple), [UXCam - Onboarding Guide](https://uxcam.com/blog/10-apps-with-great-user-onboarding/)

**Standard layout (top to bottom):**
1. `[Continue with Apple]` — black button with Apple logo (required first per App Store Review Guidelines 4.8)
2. `[Continue with Google]` — white/outline button with Google logo
3. `[Continue with Facebook]` — blue button with Facebook logo
4. `[Sign up with email or phone]` — text link or secondary button
5. Separator: "or" divider line
6. `[Log in]` text link at bottom

**Apple requirement:** If app offers third-party social sign-in, Sign in with Apple must be offered and placed at the top or equally prominent position.

**Common interaction:**
- Social buttons trigger system-level authentication (ASAuthorizationController for Apple)
- Loading spinner replaces button text during authentication
- Error: inline below button or alert
- Success: auto-transitions to next step or home screen

---

## 5. Navigation Patterns

### 5.1 Instagram Navigation (2025 Update)

**Source:** [Inro Social - Instagram Tabs Layout](https://www.inro.social/blog/instagram-tabs-new-layout-2025), [Minter.io - Instagram Navigation Update](https://minter.io/blog/instagram-launches-new-app-navigation-system-update/), [Social Media Today - Instagram DM Button](https://www.socialmediatoday.com/news/instagram-tests-placement-dm-button-in-the-main-ui/733842/)

**Tab bar order (October 2025 update):**
```
Home | Reels | DMs | Search | Profile
```

**Key changes from previous layout:**
- DMs moved to center position (was top-right icon)
- Create (+) button removed from tab bar → relocated to top-left corner
- Rationale: "Nearly all recent growth has come from DMs, Reels, and recommendations" — Adam Mosseri
- Ergonomic reasoning: thumb naturally hovers over center tabs

**Top bar (Home tab):**
- Left: Instagram logo + `[+]` create button (new position)
- Right: Notification bell (with badge count) + Likes heart
- Story bar: horizontal scroll of circular avatars below nav bar
  - Own story (first position, with + overlay if no active story)
  - Close friends' stories with green ring
  - Regular stories with gradient ring (pink/orange)
  - "Live" badge overlay for live broadcasts

**Feed navigation:**
- Pull-to-refresh
- Infinite scroll
- "New Posts" pill button appears at top when new content available
- Double-tap to like (haptic feedback)

### 5.2 TikTok Navigation

**Source:** [Tamidy - TikTok UI/UX](https://www.tamidy.com/blog/the-ui-ux-of-tiktok-first-impressions), [Iterators - TikTok UI](https://www.iteratorshq.com/blog/5-tiktok-ui-choices-that-made-the-app-successful/)

**Tab bar:**
```
Home | Friends | [+] Create | Inbox | Profile
```

- Center `[+]` button: visually distinct (taller, multicolor/white icon on black background)
- Active tab: bold label, filled icon
- Inactive tab: thin label, outline icon

**Home tab top toggles:**
- "Following" | "For You" centered at top
- "For You" is default — algorithmic feed
- "Following" — chronological from followed accounts
- Swipe left/right to toggle between them

**Video feed:**
- Full-screen immersive: video fills entire screen
- Vertical swipe (up) for next video
- Right sidebar: avatar (+follow), heart (like count), comment (count), bookmark, share
- Bottom overlay: @username, caption, #hashtags, sound name (marquee scroll)
- Tap anywhere to pause/resume

**Comments bottom sheet:**
- Tap comment icon → half-screen bottom sheet slides up
- Video continues playing behind (visible above sheet)
- Sheet expandable: drag up to full-screen
- Comments scrollable within sheet
- Reply threading with indentation
- Dismiss: swipe down on grab handle or tap outside

**Gesture-based navigation:**
- Swipe up: next video
- Swipe down: previous video (or refresh on first video)
- Swipe left: creator profile
- Swipe right: camera/create (from Home)
- Long press: speed/save/report options

### 5.3 Navigation Pattern Comparison (MyApp Applicability)

**Source:** [Frank Rausch - iOS Navigation Patterns](https://frankrausch.com/ios-navigation/)

| Pattern | Use Case | MyApp Feature |
|---|---|---|
| Tab bar (flat) | Top-level sections | Feed, Marketplace, Chat, Profile |
| Drill-down (push/pop) | Hierarchical content | Listing detail, user profile |
| Bottom sheet (low-friction) | Contextual actions | Comments, share, filters |
| Step-by-step (wizard) | Linear flows | Booking, onboarding, payment |
| Modal sheet | Self-contained tasks | Create post, edit profile |
| Pyramid (sibling swipe) | Peer content | Stories, reels, gallery images |

---

## 6. Notification Center Patterns

### 6.1 Instagram Activity Feed

**Source:** [Instagram Help - Notifications](https://help.instagram.com/124119401075803), [Kicksta - Instagram Notifications](https://blog.kicksta.co/everything-you-need-to-know-about-instagram-notifications/)

**Layout structure:**
- Full-screen activity feed accessible from heart/bell icon
- Time-based sections: "Today" | "This Week" | "This Month" | "Earlier"
- Each section has a header label

**Notification types with visual cues:**
- Like: avatar + "liked your post" + post thumbnail (right)
- Comment: avatar + "commented: [preview]" + post thumbnail
- Follow: avatar + "started following you" + `[Follow]` button (right)
- Mention: avatar + "mentioned you in a comment"
- Tag: avatar + "tagged you in a post" + post thumbnail
- Story reaction: avatar + "reacted to your story" + emoji

**Grouping pattern:**
- "username and X others liked your post" — collapsed group
- Tap to expand and see all users
- Post thumbnail on right provides context
- Unread notifications have slightly highlighted/blue background

**Follow requests section:**
- Separate "Follow Requests" link at top (for private accounts)
- Badge count on the link
- List: avatar + username + `[Confirm]` `[Delete]`

### 6.2 LinkedIn Notifications

**Source:** [Smashing Magazine - Notification UX](https://www.smashingmagazine.com/2025/07/design-guidelines-better-notifications-ux/)

**Layout structure:**
- Notifications tab in bottom bar
- Filter tabs at top: "All" | "My Posts" | "Mentions"
- Time-based grouping within each filter

**Notification types:**
- Connection accepted: avatar + "accepted your invitation"
- Post engagement: "X people viewed your post"
- Mention: avatar + "mentioned you in a post"
- Job alert: briefcase icon + job title + company
- Network update: avatar + "started a new position at..."
- Content suggestion: "Trending in your network"

**Notification management:**
- Three-dot menu per notification: Turn off, Delete, Report
- "Manage notification preferences" link
- Granular settings: per-type (Connections, Messages, Jobs, News, etc.)
- Frequency options per type: On, Off, Weekly Digest

### 6.3 Notification UX Best Practices (Cross-Platform)

**Source:** [Smashing Magazine - Notification UX](https://www.smashingmagazine.com/2025/07/design-guidelines-better-notifications-ux/), [NN/g - Push Notifications](https://www.nngroup.com/articles/push-notification/)

**Severity levels:**
1. High attention: alerts, errors, confirmations (system-level push + in-app)
2. Medium attention: warnings, acknowledgments (in-app + optional push)
3. Low attention: informational, badges (in-app only)

**Grouping strategies:**
- By type: all likes together, all comments together
- By entity: all activity on a specific post grouped
- By actor: "User X liked 3 of your photos" (Instagram approach)
- Time-windowed batching: aggregate notifications within N-minute windows

**Design rules:**
- Every notification answers: Who (actor) + What (action) + Why (context) — readable in 2 seconds
- Human-generated notifications always higher priority than automated
- Preset modes: "Calm" (low frequency) | "Regular" | "Power user" (all notifications)
- Snooze/pause: temporary mute for N hours
- Summary mode: daily/weekly digest option
- Contextual permission: ask for push notification permission when user would benefit (after first meaningful interaction, not on first launch)

---

## 7. Key Takeaways for MyApp

### Profile (Social + marketplace app)
- **Adopt Instagram's compact header pattern**: avatar (left) + stats bar (right) — proven for content-creator profiles
- **Stats bar**: Posts | Followers | Following (tappable, abbreviate at 10K+)
- **Tab switching**: Posts grid | Reels/Videos | Saved/Collections — sticky tabs on scroll
- **Verified/Creator badge**: inline after username (differentiates a creator from a standard user role)
- **Edit profile**: modal sheet with fields in order: Photo, Name, Bio, Website, Pronouns

### Social Graph (Follow + Friendship Model)
- **Follow button**: `[Follow]` (filled) → `[Following]` (outline) — unidirectional, no approval
- **Friendship button**: `[Connect]` (filled) → `[Pending]` (outline) → `[Friends]` (outline) — bidirectional, requires acceptance (LinkedIn model)
- **Combined display**: show both Follow and Connect options, similar to LinkedIn's Follow + Connect coexistence
- **Mutual followers**: "Followed by [avatar] and X others" on profile (Instagram pattern)
- **Suggestions**: "People you may know" horizontal carousel with mutual connection counts

### Marketplace (Listings)
- **Listing card**: image carousel + heart overlay + title + location + rating + price per person
- **Price display**: "From $X,XXX /person" (Airbnb-style bold price + unit)
- **Booking flow**: Step-by-step wizard — Select dates → Select guests → Review → Pay (Airbnb pattern)
- **Date picker**: Calendar range selection with price per night/date, inline or modal
- **Guest selector**: Stepper controls with category labels (Adults, Children)
- **Filters**: full-screen modal with live result count, category chips horizontal scroll
- **Map**: price-labeled pins, tap for mini card, drag to refresh (Airbnb pattern)
- **Reviews**: numeric rating + category breakdown + guest-type filter (Booking.com pattern)

### Onboarding
- **Browse first, register later**: allow feed + marketplace browsing without account (Airbnb + TikTok pattern)
- **Sign-up trigger**: when user tries to interact (like, comment, book, save)
- **Social sign-in order**: Apple (required first) → Google → Facebook → email/phone
- **Interest selection**: full-screen grid of large category buttons (TikTok pattern) — Travel, Food, Adventure, Culture, Beach, etc.
- **Permission requests**: progressive — notification on first Inbox visit, camera on first Create, location on first map view
- **30 words per step maximum** (Instagram principle)

### Navigation
- **Tab bar**: Home (Feed) | Explore | [+] Create | Marketplace | Profile
- **Center create button**: visually distinct, larger (TikTok pattern)
- **Story bar**: horizontal avatar scroll at top of feed (Instagram pattern)
- **DMs**: accessible from top-right icon on Home (pre-2025 Instagram pattern, more appropriate for MyApp's non-DM-centric model)
- **Notifications**: bell icon with badge count in top bar

### Notifications
- **Activity feed**: time-sectioned (Today, This Week, Earlier)
- **Grouping**: "X and N others liked your Post" with thumbnail
- **Types**: likes, comments, follows, friendship requests, bookings, payments
- **Mark as read**: background highlight for unread, auto-clear on view
- **Settings**: granular per-type toggles + frequency modes

---

## Sources

### Primary Sources (Official Documentation & Engineering Blogs)
- [Airbnb Engineering - Server-Driven UI](https://medium.com/airbnb-engineering/a-deep-dive-into-airbnbs-server-driven-ui-system-842244c5f5)
- [Airbnb Engineering - Map Search Ranking](https://medium.com/airbnb-engineering/improving-search-ranking-for-maps-13b03f2c2cca)
- [Airbnb Map Platform - Adam Shutsa](https://adamshutsa.com/map-platform/)
- [Instagram Help Center - Notifications](https://help.instagram.com/124119401075803)
- [Instagram Help Center - Verified Badges](https://help.instagram.com/854227311295302)
- [LinkedIn Help - Follow and Connect](https://www.linkedin.com/help/linkedin/answer/a702683)
- [LinkedIn Help - Connections You May Know](https://www.linkedin.com/help/linkedin/answer/a553108)
- [Apple HIG - Sign in with Apple](https://developer.apple.com/design/human-interface-guidelines/sign-in-with-apple)
- [Apple HIG - Notifications](https://developer.apple.com/design/human-interface-guidelines/notifications)
- [Meta Verified](https://www.meta.com/meta-verified/)
- [Booking.com Help - Using Search Filters](https://www.airbnb.com/help/article/479)

### UX Research & Case Studies
- [NN/g - Bottom Sheets](https://www.nngroup.com/articles/bottom-sheet/)
- [NN/g - Push Notifications](https://www.nngroup.com/articles/push-notification/)
- [Smashing Magazine - Notification UX Guidelines](https://www.smashingmagazine.com/2025/07/design-guidelines-better-notifications-ux/)
- [Frank Rausch - Modern iOS Navigation Patterns](https://frankrausch.com/ios-navigation/)
- [Baymard Institute - Airbnb UX Benchmark](https://baymard.com/ux-benchmark/case-studies/airbnb)
- [UI Patterns - Follow Pattern](https://ui-patterns.com/patterns/follow)

### Design Blogs & Analysis
- [Eleken - Profile Page Design Examples](https://www.eleken.co/blog-posts/profile-page-design)
- [It's Nice That - Airbnb App Redesign](https://www.itsnicethat.com/articles/airbnb-app-redesign-140525)
- [Airbnb Summer 2025 Update](https://medium.com/design-bootcamp/airbnb-summer-2025-update-heres-what-s-new-and-why-it-matters-0ced2338b921)
- [LinkedIn UI Case Study](https://medium.com/@vlad.derdeicea/linkedin-ui-case-study-ad356d131837)
- [Tamidy - TikTok UI/UX](https://www.tamidy.com/blog/the-ui-ux-of-tiktok-first-impressions)
- [Iterators - TikTok UI Choices](https://www.iteratorshq.com/blog/5-tiktok-ui-choices-that-made-the-app-successful/)
- [Appcues - TikTok Onboarding](https://goodux.appcues.com/blog/tiktok-user-onboarding)
- [Appcues - Instagram Business Onboarding](https://goodux.appcues.com/blog/instagrams-business-onboarding)
- [GoodUI - Airbnb Save Placement A/B Test](https://goodui.org/leaks/airbnb-a-b-tests-and-detects-a-better-placement-for-saving-properties/)

### UX Pattern Libraries
- [Mobbin - LinkedIn Connect Flow](https://mobbin.com/explore/flows/6bf07322-45de-4631-8d91-55c20fe1a9b1)
- [Mobbin - Instagram Onboarding](https://mobbin.com/explore/flows/fc508ec1-6af0-46bd-9b01-578751475faa)
- [Mobbin - TikTok Onboarding](https://mobbin.com/explore/flows/dfdde1ec-9e25-45fd-adf2-a160a7cf17ef)
- [Page Flows - Airbnb](https://pageflows.com/ios/products/airbnb/)
- [Page Flows - Instagram Onboarding](https://pageflows.com/post/ios/onboarding/instagram/)

### Navigation Updates (2025)
- [Inro Social - Instagram New Tab Layout](https://www.inro.social/blog/instagram-tabs-new-layout-2025)
- [Minter.io - Instagram Navigation Update](https://minter.io/blog/instagram-launches-new-app-navigation-system-update/)
- [Social Media Today - Instagram DM Button Placement](https://www.socialmediatoday.com/news/instagram-tests-placement-dm-button-in-the-main-ui/733842/)
- [ClickAnalytic - Instagram New UI 2025](https://www.clickanalytic.com/instagram-new-ui-2025-explained/)
