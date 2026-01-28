# Frontend Features Documentation

This document provides a comprehensive overview of all features implemented in the Event Reconciliation Dashboard frontend.

## Table of Contents

1. [Dashboard Overview](#dashboard-overview)
2. [Summary Cards](#summary-cards)
3. [Charts and Visualizations](#charts-and-visualizations)
4. [Data Table/Grid](#data-tablegrid)
5. [Filtering System](#filtering-system)
6. [Review and Actions](#review-and-actions)
7. [SQL Query Panel](#sql-query-panel)
8. [UI/UX Features](#uiux-features)
9. [Theme Support](#theme-support)

---

## Dashboard Overview

The Event Reconciliation Dashboard provides a comprehensive interface for managing and reviewing event matches between StubHub and Dice.fm platforms.

### Key Components

- **Header**: Displays dashboard title, description, and theme toggle
- **Summary Section**: Overview cards and charts showing match statistics
- **Data Table**: Detailed reconciliation records with filtering and sorting
- **Error Display**: Fixed error notification area at the bottom of the page

### Auto-refresh

- Summary data automatically refreshes every 30 seconds
- Real-time updates ensure data stays current

---

## Summary Cards

Interactive summary cards provide quick access to filtered views of reconciliation data.

### Card Types

1. **Total Records**
   - Shows the total count of all reconciliation records
   - Blue color scheme
   - Clicking toggles between all records and filtered view

2. **Matched**
   - Displays count and percentage of matched events
   - Green color scheme
   - Shows percentage as subtitle
   - Clicking filters to show only matched records

3. **Needs Review**
   - Shows count and percentage of records requiring review
   - Yellow color scheme
   - Clicking filters to show only records needing review

4. **Unmatched**
   - Displays combined count of unmatched StubHub and unmatched Dice events
   - Red color scheme
   - Shows combined percentage
   - Clicking filters to show all unmatched records

### Features

- **Radio Button Behavior**: Only one card can be selected at a time
- **Visual Feedback**: Selected cards show ring border, scale effect, and "Filter active" indicator
- **Hover Effects**: Cards have hover animations and scale effects
- **Dark Mode Support**: Full dark mode styling

---

## Charts and Visualizations

### Match Status Pie Chart

An interactive pie chart displaying the distribution of match statuses.

#### Features

- **External Labels**: Labels positioned outside the pie with arrows pointing to slices
- **Interactive Slices**: Clicking a slice filters the table to that match status
- **Selection Highlighting**: Selected slice is highlighted with blue border
- **Grey Unselected**: When a slice is selected, all other slices turn grey
- **Clickable Legend**: Legend buttons below the chart allow filtering
- **Color Coding**:
  - Matched: Green (#10b981)
  - Needs Review: Yellow (#f59e0b)
  - Unmatched StubHub: Red (#ef4444)
  - Unmatched Dice: Dark Red (#b91c1c)

#### Visual Elements

- Custom label lines with arrow markers
- Percentage display on labels
- Responsive container that adapts to screen size
- Tooltip on hover showing detailed information

### Confidence Breakdown Card

Displays match confidence distribution with color-coded badges.

#### Features

- **Color Consistency**: Badge colors match the confidence badges in the data grid
- **Filter Integration**: Clicking a confidence level filters the table
- **Excludes Unmatched**: Unmatched records (confidence 0) are not included in breakdown
- **Auto-clear**: Selecting "Unmatched" match status automatically clears confidence filter

---

## Data Table/Grid

The main reconciliation table provides detailed record information with extensive filtering, sorting, and interaction capabilities.

### Column Structure

1. **Selection Checkbox**: Multi-select records for bulk operations
2. **Links**: Quick access to StubHub and Dice event URLs
3. **ID**: Internal reconciliation record ID
4. **Status**: Match status badge (Matched, Needs Review, Unmatched)
5. **StubHub Event**: Event name, date, time, and quick ID filter
6. **Dice Event**: Event name, date, and ID
7. **Venue**: Venue name and location
8. **Score**: Name matching score
9. **Confidence**: Match confidence badge with color coding
10. **Review**: Review status dropdown
11. **Crosswalk**: Multi-line display of mapping ID, StubHub ID, and Dice ID
12. **Actions**: Individual record actions

### Table Features

#### Column Filtering

- **Filter Icons**: Each column header has a filter icon button
- **Filter Modal**: Clicking opens a modal with filter options specific to that column
- **Active Filter Indicators**: Filter icons highlight when filters are active
- **Column-Specific Filters**:
  - **Match Status**: Radio buttons (All, Matched, Needs Review, Unmatched)
  - **Confidence**: Multi-select checkboxes
  - **Review Status**: Multi-select checkboxes
  - **Venue**: City filter integration, venue name search
  - **Crosswalk**: Radio buttons (All, Matched, Mismatched, Unlinked)
  - **Date Ranges**: Date pickers for StubHub and Dice event dates
  - **Score Range**: Min/max score inputs

#### Quick Filters

- **StubHub ID Quick Filter**:
  - Search icon in StubHub Event column header
  - Expands to input field on click
  - Auto-focuses and selects text for easy typing
  - Applies filter on Enter key or blur
  - Bypasses all other filters for direct lookup
  - Icon turns green when filter is active
  - Escape key closes the input

#### Sorting

- **Column Sorting**: Click column headers to sort
- **Sort Indicators**: Up/down arrows show sort direction
- **Multi-field Sorting**: Sort by any column
- **Persistent Sorting**: Sort state maintained during navigation

#### Pagination

- **Page Navigation**: Previous/Next buttons
- **Page Size**: Configurable records per page (default: 50)
- **Page Info**: Displays current page, total pages, and record counts
- **Auto-reset**: Page resets to 1 when filters change

#### Selection

- **Individual Selection**: Click checkboxes to select individual records
- **Select All**: Header checkbox selects/deselects all visible records
- **Bulk Actions**: Selected records enable bulk review and approve to crosswalk actions
- **Selection Counter**: Shows count of selected records

#### Column Resizing

- **Resizable Columns**: Drag column borders to adjust widths
- **Persistent Widths**: Column widths saved to localStorage
- **Default Widths**: Pre-configured default column widths
- **Visual Feedback**: Resize cursor and highlight during resize

#### Grouping

- **Group By Options**: 
  - Venue
  - Date
  - City
  - Match Status
  - **StubHub Event ID** (groups by `stubhub_id`)
  - **Dice Event ID** (groups by `dice_event_dice_id`)
- **Expandable Groups**: Click to expand/collapse groups
- **Group Headers**: Show aggregate information per group
- **Group Labels**: 
  - StubHub ID groups display as "StubHub ID: {id}"
  - Dice ID groups display as "Dice ID: {id}"

#### Data Display

- **Multi-line Crosswalk**: Crosswalk column displays:
  - Mapping ID (first line)
  - StubHub ID (second line)
  - Dice ID (third line)
- **Crosswalk Cell Background Colors**:
  - **Green Background**: When crosswalk `stubhub_id` matches event `stubhub_id` AND crosswalk `dice_id` matches event `dice_id`
  - **Red Background**: When crosswalk exists but IDs don't match
  - **No Background**: When there's no crosswalk entry (mapping_id is null)
- **Status Badges**: Color-coded badges for match status and confidence
- **Date Formatting**: Human-readable date formats
- **URL Links**: Clickable links to external event pages
- **Empty State**: "No records found" message when filters return no results
- **Header Persistence**: Table header remains visible even with no records

---

## Filtering System

The dashboard provides multiple layers of filtering for precise data exploration.

### Filter Types

#### 1. Match Status Filter

- **Location**: Summary cards and pie chart
- **Options**: All, Matched, Needs Review, Unmatched
- **Behavior**: Radio button (single selection)
- **Integration**: Filters both summary and table data

#### 2. Confidence Filter

- **Location**: Confidence breakdown card
- **Options**: All confidence levels (High, Medium, Low)
- **Behavior**: Multi-select
- **Auto-exclude**: Unmatched records (confidence 0) excluded
- **Auto-clear**: Clears when "Unmatched" match status selected

#### 3. Crosswalk Filter

- **Location**: Dashboard filter bar and column header
- **Options**: All, Matched, Mismatched, Unlinked
- **Behavior**: Radio buttons
- **Definition**:
  - **Matched**: Records with exact (stubhub_id, dice_id) in crosswalk_stubhub
  - **Mismatched**: Crosswalk row exists but IDs differ from event
  - **Unlinked**: Records without mapping_id

#### 4. City/Location Filter

- **Location**: Location filter bar at top of dashboard
- **Options**: 
  - Default cities only (toggle)
  - Individual city selection (multi-select)
- **Impact**: Affects all statistics and charts
- **Integration**: Works with venue column filter

#### 5. Column Filters

- **Per-column Filtering**: Each column has its own filter modal
- **Filter Persistence**: Filter values preserved when modal reopens
- **Smart Initialization**: Uses selected record values when available
- **Minimize/Restore**: Filter modal can be minimized to save space

#### 6. Search Filter

- **Text Search**: Searches event names and venue names
- **Numeric Search**: Direct stubhub_id lookup (bypasses all other filters)
- **Location**: Global search or column-specific

#### 7. Date Range Filters

- **StubHub Events**: Filter by StubHub event date range
- **Dice Events**: Filter by Dice event date range
- **Date Pickers**: User-friendly date input fields
- **Format**: YYYY-MM-DD input, stored as YYYYMMDD

#### 8. Score Range Filters

- **Min Score**: Minimum name matching score
- **Max Score**: Maximum name matching score
- **Numeric Input**: Number input fields with validation

### Filter Behavior

- **Combined Filters**: Multiple filters work together (AND logic)
- **Filter Priority**: Column filters take precedence over dashboard filters
- **StubHub ID Override**: Numeric stubhub_id search bypasses all other filters
- **Active Filter Display**: Shows all active filters in a summary bar
- **Clear Filters**: Individual and bulk clear options
- **Filter Reset**: Reset button in filter modals

---

## Review and Actions

### Individual Record Review

#### Review Dropdown

- **Location**: Review column in data table
- **Options**: 
  - Pending
  - Approved
  - Rejected
  - Needs Review
- **Real-time Update**: Changes reflected immediately
- **Status Badge**: Visual indicator of review status

#### Review Modal

- **Detailed Review**: Full record details in modal
- **Review Notes**: Optional notes field
- **Reviewer Tracking**: Records who reviewed and when
- **Status Update**: Change review status from modal

### Bulk Actions

#### Bulk Review Panel

- **Trigger**: Appears when records are selected
- **Features**:
  - Review multiple records at once
  - Set review status for all selected
  - Add review notes
  - Track reviewer
- **Validation**: Ensures at least one record selected

#### Approve to Crosswalk

- **Purpose**: Move approved records to crosswalk_stubhub table
- **Trigger**: "Approve to Crosswalk" button when records selected
- **Features**:
  - Asynchronous processing (202 response pattern)
  - Job status polling
  - Progress indication
  - Success/failure notifications
  - Reviewer and notes tracking
- **Workflow**:
  1. Select records to approve
  2. Click "Approve to Crosswalk"
  3. Enter reviewer name and optional notes
  4. Submit (returns immediately with job ID)
  5. Background processing begins
  6. Status polled until completion
  7. Results displayed (processed, upserted, skipped, failed counts)
- **Requirements**: Only approved records are moved to crosswalk
- **Auto-refresh**: Table and summary refresh after completion

---

## SQL Query Panel

The SQL Query Panel provides visibility into the database queries executed for each API request, along with curl command equivalents for easy testing and debugging.

### Features

- **Optional Display**: Panel is hidden by default and only appears when explicitly opened
- **On-Demand Tracking**: SQL queries are only tracked when the panel is opened (via `include_queries=true` parameter)
- **Request Hashcode**: Each API response includes a unique hashcode for tracking and verification
- **Data Integrity**: Hashcode validation ensures queries match the data they belong to

### Panel Components

#### Toggle Button

- **Location**: Below the reconciliation table
- **Always Visible**: Button is always visible, even when queries aren't loaded
- **Display**: Shows "SQL Queries" with optional count when queries are available
- **Hashcode Display**: Shows request hashcode in the button for quick reference
- **Click to Open**: Opens panel and triggers query tracking for the current request

#### Query Display

- **Formatted SQL**: Shows SQL with parameters substituted for readability
- **Raw SQL**: Shows original SQL with `?` placeholders
- **Parameters**: Displays parameter values as formatted JSON
- **Query Duration**: Shows execution time for each query in milliseconds
- **Copy to Clipboard**: One-click copy for each query

#### cURL Command

- **Auto-generated**: Automatically generates curl command for the API call
- **Complete Request**: Includes all query parameters and headers
- **Copy to Clipboard**: Easy copy for testing in terminal
- **Ready to Use**: Can be pasted directly into terminal

#### Request Hashcode Info

- **Hashcode Display**: Shows the unique request hashcode
- **Verification**: Validates that queries match the data by comparing hashcodes
- **Warning System**: Displays warning if hashcodes don't match (prevents data bleeding)
- **Tracking**: Hashcode can be used to correlate requests in logs

### Technical Details

- **Performance**: No overhead when panel is closed (queries not tracked)
- **Request Hashcode**: Generated using MD5 hash of query parameters, timestamp, and random value
- **Hashcode Format**: 16-character hexadecimal string (e.g., `a3f2b1c4d5e6f7a8`)
- **Query Tracking**: Only active when `include_queries=true` is in the request
- **Response Structure**: 
  - All responses include `requestHash`
  - Query responses include `queries` array and `queriesMeta` with hashcode verification

### Use Cases

- **Debugging**: Understand exactly what SQL queries are executed
- **Performance Analysis**: See query execution times
- **API Testing**: Copy curl commands to test API endpoints
- **Data Verification**: Verify queries match the data using hashcodes
- **Documentation**: Document API usage with curl commands

---

## UI/UX Features

### Responsive Design

- **Mobile Support**: Responsive grid layouts
- **Breakpoints**: Tailwind CSS breakpoints (sm, md, lg, xl)
- **Flexible Layouts**: Components adapt to screen size
- **Touch-friendly**: Large touch targets for mobile

### Loading States

- **Loading Indicators**: Spinner animations during data fetch
- **Skeleton Screens**: Placeholder content while loading
- **Progress Feedback**: Visual feedback for long operations

### Error Handling

- **Error Display**: Fixed error notification area at bottom
- **Error Details**: Expandable error details
- **Error Dismissal**: Close button for individual errors
- **Error Persistence**: Errors remain until dismissed
- **Error Prevention**: Duplicate error suppression (5-second window)

### Accessibility

- **Keyboard Navigation**: Full keyboard support
- **ARIA Labels**: Screen reader support
- **Focus Management**: Proper focus handling
- **Color Contrast**: WCAG compliant color schemes
- **Semantic HTML**: Proper HTML structure

### User Feedback

- **Hover Effects**: Visual feedback on interactive elements
- **Click Feedback**: Scale and shadow effects on clicks
- **Active States**: Clear indication of selected/active items
- **Tooltips**: Helpful tooltips on icons and buttons
- **Status Messages**: Success and error notifications

### Performance

- **React Query**: Efficient data fetching and caching
- **Pagination**: Load only visible records
- **Debouncing**: Filter input debouncing (where applicable)
- **Memoization**: Component memoization for performance
- **Lazy Loading**: Components loaded on demand

### Data Persistence

- **LocalStorage**: Column widths, preferences saved
- **URL State**: Filter state could be synced with URL (future)
- **Session Persistence**: Maintains state during session

---

## Theme Support

### Dark Mode

- **Toggle Button**: Theme toggle in header
- **System Preference**: Respects system theme preference
- **Manual Override**: User can manually toggle theme
- **Persistent**: Theme preference saved

### Theme Features

- **Full Coverage**: All components support dark mode
- **Color Schemes**: 
  - Light: White/gray backgrounds, dark text
  - Dark: Dark gray backgrounds, light text
- **Transitions**: Smooth color transitions when switching themes
- **Consistent Styling**: All UI elements themed consistently

### Color Palette

#### Light Mode
- Background: Gray-50
- Cards: White
- Text: Gray-900
- Borders: Gray-200

#### Dark Mode
- Background: Gray-900
- Cards: Gray-800
- Text: Gray-100
- Borders: Gray-700

---

## Technical Implementation

### Technology Stack

- **React**: UI framework
- **TypeScript**: Type safety
- **TanStack Query (React Query)**: Data fetching and state management
- **TanStack React Table**: Data grid component
- **Recharts**: Chart library
- **Tailwind CSS**: Styling
- **Lucide React**: Icons
- **Vite**: Build tool

### Grid Architecture

Grids follow the **schema-driven architecture** defined in `docs/specs/TANSTACK_GRID_SPEC.md`:

- **SchemaGrid component**: Universal grid that consumes JSON schemas
- **Grid schemas**: Declarative configuration defining columns, filters, actions
- **Cell renderers**: Reusable components registered by name
- **AI-centric design**: Flat, explicit properties optimized for AI modifications

See Module 6 of `TANSTACK_GRID_SPEC.md` for complete documentation.

### Component Architecture

- **Component Hierarchy**:
  ```
  DashboardLayout
    └── Dashboard
        ├── LocationFilterBar
        ├── SummaryCards
        ├── MatchStatusChart
        └── ReconciliationTable
            ├── FilterModal
            ├── ReviewDropdown
            ├── BulkReviewPanel
            └── ApproveToCrosswalkPanel
  ```

### State Management

- **React Query**: Server state management
- **React State**: Local component state
- **Context API**: Theme context
- **LocalStorage**: User preferences

### API Integration

- **Service Layer**: `reconciliationService.ts` handles all API calls
- **Error Handling**: Centralized error handling
- **Type Safety**: Full TypeScript types for API responses
- **Request/Response**: Proper request/response handling

---

## Future Enhancements

Potential features for future development:

1. **Export Functionality**: Export filtered data to CSV/Excel
2. **Advanced Sorting**: Multi-column sorting
3. **Saved Filters**: Save and reuse filter combinations
4. **URL State Sync**: Sync filter state with URL parameters
5. **Keyboard Shortcuts**: Power user keyboard shortcuts
6. **Bulk Export**: Export selected records
7. **Record Comparison**: Side-by-side record comparison
8. **History Tracking**: View record change history
9. **Comments/Notes**: Per-record comments system
10. **Notifications**: Real-time notifications for job completion

---

## Summary

The Event Reconciliation Dashboard frontend provides a comprehensive, user-friendly interface for managing event reconciliation data. With extensive filtering, sorting, and review capabilities, users can efficiently process large volumes of reconciliation records while maintaining data quality and accuracy.

Key strengths:
- **Intuitive UI**: Clean, modern interface with clear visual hierarchy
- **Powerful Filtering**: Multiple filter types for precise data exploration
- **Efficient Workflows**: Bulk actions and quick filters speed up common tasks
- **Responsive Design**: Works well on all screen sizes
- **Accessibility**: Full keyboard and screen reader support
- **Performance**: Optimized for large datasets
