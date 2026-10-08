=============================================================================
=====              ConnectWise API SLA Inspector Blueprint             ======
=============================================================================

# --- Configuration ---
# Replace these with your actual ConnectWise API credentials
COMPANY_ID = "your_company"
PUBLIC_KEY = "your_public_key"
PRIVATE_KEY = "your_private_key"
BASE_URL = "https://connectwise.com" # Adjust region prefix if needed

# Setup Authorization Header (Basic Auth using Company+Public:Private)
# ConnectWise expects: base64(companyid + '+' + publickey + ':' + privatekey)
# requests can handle basic auth tuple directly if parsed out, or format string:
auth_string = (COMPANY_ID, f"{PUBLIC_KEY}:{PRIVATE_KEY}")

headers = {
    "ClientId": "your-registered-client-id", # Request a Client ID from CW Developer Network
    "Accept": "application/json"
}

def inspect_ticket_sla(ticket_id):
    """
    Fetches ticket details and audit notes from ConnectWise Manage 
    to evaluate SLA response time compliance.
    """
    # 1. Fetch Ticket Information
    ticket_url = f"{BASE_URL}/service/tickets/{ticket_id}"
    ticket_response = requests.get(ticket_url, auth=auth_string, headers=headers)
    
    if ticket_response.status_code != 200:
        print(f"[-] Failed to fetch ticket {ticket_id}: {ticket_response.text}")
        return

    ticket_data = ticket_response.json()
    date_entered_str = ticket_data.get("dateEntered") # e.g., "2026-10-07T14:30:00Z"
    
    # 2. Fetch Ticket Time Entries / Audit Log to find the first internal touchpoint
    time_entries_url = f"{BASE_URL}/service/tickets/{ticket_id}/timeentries"
    entries_response = requests.get(time_entries_url, auth=auth_string, headers=headers)
    
    first_touch_date = None
    if entries_response.status_code == 200:
        entries = entries_response.json()
        # Sort entries chronologically by actual start time
        if entries:
            entries.sort(key=lambda x: x.get("timeStart", ""))
            first_touch_date_str = entries[0].get("timeStart")
            if first_touch_date_str:
                first_touch_date = datetime.strptime(first_touch_date_str[:19], "%Y-%m-%dT%H:%M:%S")

    # 3. Analyze Compliance
    date_entered = datetime.strptime(date_entered_str[:19], "%Y-%m-%dT%H:%M:%S")
    
    print(f"\n[+] Inspecting Ticket #{ticket_id}: {ticket_data.get('summary')}")
    print(f"    - Date Created: {date_entered}")
    
    if first_touch_date:
        response_time_delta = first_touch_date - date_entered
        minutes_to_respond = response_time_delta.total_seconds() / 60
        print(f"    - First Action Taken: {first_touch_date}")
        print(f"    - Actual Response Time: {minutes_to_respond:.1f} minutes")
        
        # Example Threshold Check: Proper Sky's benchmark is ~4 minutes
        if minutes_to_respond > 15.0: # Assuming your target SLA is 15 mins
            print("    ⚠️ SLA Status: BREACHED (Exceeded response threshold)")
        else:
            print("    ✅ SLA Status: COMPLIANT")
    else:
        print("    ⚠️ SLA Status: UNTOUCHED (No time logs found on this open ticket)")

if __name__ == "__main__":
    # Test with a known ConnectWise ticket ID
    sample_ticket_id = 123456
    inspect_ticket_sla(sample_ticket_id)


=============================================================================
=====            Batch SLA Inspector Blueprint with Pagination         ======
=============================================================================

# --- Configuration ---
COMPANY_ID = "your_company"
PUBLIC_KEY = "your_public_key"
PRIVATE_KEY = "your_private_key"
BASE_URL = "https://connectwise.com"

auth_string = (COMPANY_ID, f"{PUBLIC_KEY}:{PRIVATE_KEY}")
headers = {
    "ClientId": "your-registered-client-id",
    "Accept": "application/json"
}

# --- SLA Parameters ---
# Replace this with the name of the service board you want to scan
TARGET_BOARD = "Help Desk" 
# Your target response SLA in minutes (e.g., Proper Sky's internal goal is sub-4 mins)
SLA_TARGET_MINUTES = 15.0  

def get_open_tickets_from_board(board_name):
    """
    Uses ConnectWise pagination loops to fetch ALL open tickets from a specific service board.
    """
    tickets = []
    page = 1
    page_size = 100  # Max chunk per request allowed by ConnectWise
    
    # CW conditions syntax: board/name matches and closedFlag is false
    conditions = f"board/name='{board_name}' and closedFlag=false"
    
    print(f"[+] Scanning service board '{board_name}' for open tickets...")
    
    while True:
        # Build url with pagination query filters
        url = f"{BASE_URL}/service/tickets?conditions={conditions}&pageSize={page_size}&page={page}"
        response = requests.get(url, auth=auth_string, headers=headers)
        
        if response.status_code != 200:
            print(f"[-] Error fetching page {page}: {response.text}")
            break
            
        data = response.json()
        if not data:
            break # Break loop if page returns empty (we reached the end)
            
        tickets.extend(data)
        page += 1
        
    print(f"[+] Total open tickets retrieved for evaluation: {len(tickets)}")
    return tickets

def evaluate_ticket_response(ticket):
    """
    Calculates the time elapsed since the ticket was entered 
    and checks if it has breached the SLA response matrix.
    """
    ticket_id = ticket.get("id")
    summary = ticket.get("summary", "No Summary")
    date_entered_str = ticket.get("dateEntered")
    
    if not date_entered_str:
        return
        
    date_entered = datetime.strptime(date_entered_str[:19], "%Y-%m-%dT%H:%M:%S")
    
    # Check if there is an initial response timestamp logged directly on the ticket.
    # ConnectWise sometimes tracks 'dateResponded' if standard status workflows are mapped.
    date_responded_str = ticket.get("dateResponded")
    
    if date_responded_str:
        date_responded = datetime.strptime(date_responded_str[:19], "%Y-%m-%dT%H:%M:%S")
        elapsed_minutes = (date_responded - date_entered).total_seconds() / 60
        
        if elapsed_minutes > SLA_TARGET_MINUTES:
            print(f"🚨 [BREACHED] Ticket #{ticket_id} ({summary[:40]}...): Responded in {elapsed_minutes:.1f} mins (Target: {SLA_TARGET_MINUTES}m)")
        else:
            print(f"✅ [COMPLIANT] Ticket #{ticket_id}: Responded in {elapsed_minutes:.1f} mins.")
            
    else:
        # If no explicit response date exists yet, see how long it's been sitting completely open
        current_age_minutes = (datetime.utcnow() - date_entered).total_seconds() / 60
        if current_age_minutes > SLA_TARGET_MINUTES:
            print(f"🔥 [CRITICAL BREACH] Ticket #{ticket_id} ({summary[:40]}...) has been UNTOUCHED for {current_age_minutes:.1f} mins!")
        else:
            print(f"⏳ [PENDING] Ticket #{ticket_id}: Created {current_age_minutes:.1f} mins ago. Still within SLA window.")

if __name__ == "__main__":
    open_tickets = get_open_tickets_from_board(TARGET_BOARD)
    
    print("\n--- Running SLA Auditing Logic ---")
    for ticket in open_tickets:
        evaluate_ticket_response(ticket)

