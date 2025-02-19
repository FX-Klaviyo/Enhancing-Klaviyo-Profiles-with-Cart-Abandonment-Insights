# Enhancing-Klaviyo-Profiles-with-Cart-Abandonment-Insights

### **Summary of Functionality**

* **Triggered by an event**: The Lambda function processes an event containing a `ProductID` and `profile_id`.  
* **Fetches the user profile**: Calls Klaviyo’s API to get the current profile data.  
* **Updates `Removed_IDs` list**: Adds the `ProductID` if it’s not already present.  
* **Handles errors**: Returns appropriate HTTP status codes based on missing or invalid data.

### **Use Cases**

* When a user **removes a product** from their wishlist or cart.  
* When tracking **user preferences** for excluding certain products.  
* For **personalized email marketing**, avoiding sending recommendations for products a user has removed.  
* To understand a customer's behaviour with Products, a Profile might have decided to remove the item because it is too expensive (especially if the Product is unique to this brand), maybe an opportunity to offer a discount for this item alone. 

### **Instructions**

1. Create a flow triggered by the Removed Cart event that you can create using the snippet you can find in [Klaviyo\_Shopify\_Cart\_tracking\_Enhancement](https://github.com/FX-Klaviyo/Klaviyo_Shopify_Cart_tracking_Enhancement/)  
2. Once you have created this flow you can follow the step explained in [Add a custom action to a flow](https://developers.klaviyo.com/en/docs/add_a_custom_action_to_a_flow)   
3. First, you need to add our modules and authentication  
   1. Click “modules” \> “add new module” \> “requests”  
   2. Click “modules” \> “add new module” \> “klaviyo-api”  
   3. Click “Environment Variables” \> add a new variable called “API\_KEY” and input a private API key. Note: API key must have the following permissions: Coupons: read, Coupons:write, Events:write  
4. Click “Editor” and add the following code into the editor:

```py
from klaviyo_api import KlaviyoAPI
import os
import json

# Initialize Klaviyo API client using API key from environment variables
client = KlaviyoAPI(os.getenv('API_KEY'))

# Function to update a user's profile by adding a ProductID to the 'Removed_IDs' list
def update_profile(profile_id, product_id):
    try:
        # Fetch the current profile details from Klaviyo
        response = client.Profiles.get_profile(profile_id)

        # Extract profile data properly
        profile_data = response.data  # Access the data attribute
        attributes = profile_data.attributes  # Extract profile attributes
        properties = attributes.properties if attributes.properties else {}  # Ensure properties exist

        # Retrieve existing 'Removed_IDs' or initialize an empty list
        removed_ids = properties.get('Removed_IDs', [])

        # Add ProductID to the list only if it's not already included
        if product_id not in removed_ids:
            removed_ids.append(product_id)

            # Construct the correct payload for updating the profile
            update_payload = {
                "data": {
                    "type": "profile",
                    "id": profile_id,
                    "attributes": {
                        "properties": {
                            "Removed_IDs": removed_ids  # Update the 'Removed_IDs' property
                        }
                    }
                }
            }

            # Send the update request to Klaviyo
            client.Profiles.update_profile(profile_id, update_payload)
            print(f"Successfully updated profile {profile_id} with ProductID {product_id}.")
        else:
            print(f"ProductID {product_id} already exists in Removed_IDs.")

    except Exception as e:
        print(f"Error updating profile {profile_id}: {str(e)}")

# AWS Lambda handler function to process incoming event data
def handler(event, context):
    print("Event Data:", json.dumps(event, indent=2))  # Log the event for debugging

    # Extract ProductID from event data payload
    product_id = event.get("data", {}).get("attributes", {}).get("event_properties", {}).get("ProductID")
    if not product_id:
        print("No Product ID found in event data.")
        return {
            "statusCode": 400,
            "body": json.dumps("Product ID is required.")
        }

    # Extract profile_id from event data payload
    profile_id = event.get("data", {}).get("relationships", {}).get("profile", {}).get("data", {}).get("id")
    if not profile_id:
        print("No profile ID found in event data.")
        return {
            "statusCode": 400,
            "body": json.dumps("Profile ID is required.")
        }

    # Attempt to update the profile with the given ProductID
    try:
        update_profile(profile_id, product_id)
    except Exception as e:
        print(f"Error updating profile: {e}")
        return {
            "statusCode": 500,
            "body": json.dumps(f"Error updating profile: {str(e)}")
        }

    return {
        "statusCode": 200,
        "body": json.dumps("Profile update process completed.")
    }


```
