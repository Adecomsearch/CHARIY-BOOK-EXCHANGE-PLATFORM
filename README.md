# CHARITY-BOOK-EXCHANGE-PLATFORM
Books Sharing Technology Database’ and numbered Copyright No. RZ64704 has been registered in the name of ATHANAS OWAKO.

SUMMARY OF PLATFORM INSTRUCTIONS
Platform Goal
Create an SMS-based book donation and request system where:

Donors send book details via SMS to short code EXAMPLE 444786.

Beneficiaries request books via the same short code.

The system:
Matches requests to available books.
Sends SMS with donor contact/location to the requester.
Sends a “Thank You” message to the donor after a successful match.
SQL Database

SMS FORMAT
✅ Donor Sends
To: 444786
Message:
OFFER|Title|Author|Year|Grade|Ward|County
Example:
OFFER|Mathematics Essentials|John Doe|2018|Form 2|Kileleshwa|Nairobi

✅ Beneficiary Requests
To: 444786
Message:
REQUEST|Title|Grade
Example:
REQUEST|Mathematics Essentials|Form 2


PostgreSQL Components
🔹 Tables
users: Phone numbers and roles.
books_offered: Books listed for donation.
book_requests: Incoming SMS-based book requests.
Wards: Kenyan ward and county reference data.
Sms_logs: Stores incoming/outgoing SMS (optional for auditing).

🔹 Sample Nairobi Wards
sql
INSERT INTO wards (ward_name, county_name) VALUES
('Kileleshwa', 'Nairobi'),
('Kilimani', 'Nairobi'),
('Langata', 'Nairobi'),
('Westlands', 'Nairobi');

TRIGGER FUNCTION — Match + SMS + Thank You
This function:
Looks for a matching book.
Sends SMS to the beneficiary with the donor contact.
Sends thank-you SMS to the donor.

sql

CREATE OR REPLACE FUNCTION notify_sms_response()
RETURNS TRIGGER AS $$
DECLARE
    matched RECORD;
    requester_phone TEXT;
    donor_thank_you TEXT;
    beneficiary_info TEXT;
BEGIN
    SELECT 
        bo.title, bo.author, bo.edition_year, bo.academic_level,
        u.phone_number AS donor_phone, 
        w.ward_name, w.county_name
    INTO matched
    FROM books_offered bo
    JOIN users u ON bo.user_id = u.user_id
    JOIN wards w ON bo.ward_id = w.ward_id
    WHERE LOWER(bo.title) = LOWER(NEW.title)
      AND LOWER(bo.academic_level) = LOWER(NEW.academic_level)
      AND bo.available = TRUE
    LIMIT 1;

    requester_phone := (SELECT phone_number FROM users WHERE user_id = NEW.user_id);

    IF FOUND THEN
        -- Message to beneficiary
        beneficiary_info := 'Book match found: "' || matched.title || '", ' || matched.academic_level || 
                            ' (' || matched.edition_year || ') at ' || matched.ward_name || ' Ward, ' || matched.county_name ||
                            ' County. Contact: ' || matched.donor_phone;

        PERFORM pg_notify('send_sms', json_build_object(
            'to', requester_phone,
            'message', beneficiary_info
        )::TEXT);

        -- Thank you message to donor
        donor_thank_you := 'Thank you for donating "' || matched.title || '" (Grade: ' || matched.academic_level || '). ' ||
                           'Your support is helping a student in need. – Charity Book Swap Team';

        PERFORM pg_notify('send_sms', json_build_object(
            'to', matched.donor_phone,
            'message', donor_thank_you
        )::TEXT);
    ELSE
        PERFORM pg_notify('send_sms', json_build_object(
            'to', requester_phone,
            'message', 'Sorry, no matching book titled "' || NEW.title || '" for ' || NEW.academic_level || ' is available now.'
)::TEXT);
    END IF;

    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

Trigger
CREATE TRIGGER trigger_notify_on_book_request
AFTER INSERT ON book_requests
FOR EACH ROW
EXECUTE FUNCTION notify_sms_response();

Example SMS Responses
To Beneficiary:
psql
Book match found: "Mathematics Essentials", Form 2 (2018)
Location: Kileleshwa Ward, Nairobi County
Contact: +254712xxxxxx
Please reach out to arrange collection.

To Donor:
Thank you for donating "Mathematics Essentials" (Grade: Form 2).
Your support is helping a student in need.
– Charity Book Swap Team

