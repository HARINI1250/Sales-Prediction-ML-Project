import time
import re
from selenium import webdriver
from selenium.webdriver.common.by import By
from selenium.webdriver.common.keys import Keys
from selenium.webdriver.support.ui import WebDriverWait
from selenium.webdriver.support import expected_conditions as EC

# Set up the WebDriver
driver = webdriver.Chrome()

# Navigate to the webpage
driver.get("https://www.woodenstreet.com/fabric-sofas")

# Wait for the page to load
time.sleep(10)  # Adjust this based on your network speed

# Click the "Sort By" option
try:
    sort_by_element = WebDriverWait(driver, 10).until(
        EC.element_to_be_clickable((By.ID, "filter-tag"))
    )
    sort_by_element.click()
    print("Clicked the 'Sort By' option.")
except Exception as e:
    print(f"Error clicking 'Sort By' option: {e}")

# Select "Price (High to Low)"
try:
    price_high_to_low_element = WebDriverWait(driver, 10).until(
        EC.element_to_be_clickable(
            (By.XPATH, '//input[@value="p.price-DESC"]/following-sibling::span[text()=" Price (High to Low)"]'))
    )
    price_high_to_low_element.click()
    print("Selected 'Price (High to Low)'.")
except Exception as e:
    print(f"Error selecting 'Price (High to Low)': {e}")

# Click the cross mark to close the filter page
try:
    close_filter_element = WebDriverWait(driver, 10).until(
        EC.element_to_be_clickable((By.XPATH, '//div[@class="BackTo"]'))
    )
    close_filter_element.click()
    print("Clicked the cross mark to close the filter page.")
except Exception as e:
    print(f"Error clicking the cross mark: {e}")

# Wait for the products to load after closing the filter page
time.sleep(5)

# Count the number of products
try:
    products = driver.find_elements(By.XPATH, '//div[@class="col"]')
    print(f"Number of products: {len(products)}")
except Exception as e:
    print(f"Error counting the number of products: {e}")

# Loop through each product to click and open in a new tab, limit to 350 products
product_count = 0
for product in products:
    if product_count >= 350:
        break
    try:
        # Extracting number of items sold for each product
        sold_count = 0
        sold_count_element = product.find_elements(By.XPATH, './/div[@class="rating-box"]//span[@class="rating-count"]')
        if sold_count_element:
            sold_count_text = sold_count_element[0].text
            sold_count = int(''.join(filter(str.isdigit, sold_count_text))) if any(
                char.isdigit() for char in sold_count_text) else 0

        # Clicking the product image to open in a new tab
        product_image = WebDriverWait(product, 1).until(
            EC.element_to_be_clickable((By.XPATH, './/a[@class="targetBlank"]'))
        )
        product_image.send_keys(Keys.CONTROL + Keys.RETURN)  # Open link in new tab

        # Switch to the newly opened tab
        driver.switch_to.window(driver.window_handles[-1])

        # Wait for the product detail to load and extract the product name, price, and other details
        try:
            product_details = {}

            product_name_element = WebDriverWait(driver, 1).until(
                EC.presence_of_element_located((By.XPATH, '//h1[@class="product-name"]'))
            )
            product_details["Name"] = product_name_element.text

            price_element = WebDriverWait(driver, 1).until(
                EC.presence_of_element_located((By.XPATH, '//div[@class="pricedetail"]'))
            )
            product_details["Price"] = price_element.text.split("Rs")[1].strip().replace(",", "")

            detail_attributes = [
                "Designs", "Material", "Color", "Warranty", "Foam", "Seater", "Armrest",
                "Delivery Condition", "Shape", "Brand"
            ]

            dimensions_attributes_cm = [
                "Dimensions (Cm)", "Dimension of Sofa (Cm)",
                "Sofa Dimensions (Cm)", "Three Seater Dimensions (Cm)", "Dimension 1 (Cm)",
                "Outer Dimensions (Cm)", "Dimension of One Seater (Cm)", "Dimension of Three Seater (Cm)"
            ]

            for attr in detail_attributes:
                try:
                    detail_element = WebDriverWait(driver, 2).until(
                        EC.presence_of_element_located((By.XPATH,
                                                        f'//div[@class="overview"]//span[@class="product-attr" and text()="{attr}"]/following-sibling::span[@class="product-value"]'))
                    )
                    if attr == "Warranty":
                        text = detail_element.text
                        product_details[attr] = int(''.join(filter(str.isdigit, text))) if any(
                            char.isdigit() for char in text) else 0
                    elif attr == "Seater":
                        text = detail_element.text
                        if '+' in text:
                            parts = text.split('+')
                            total_seater = sum(int(part.strip()) for part in parts if part.strip().isdigit())
                            product_details[attr] = total_seater
                        else:
                            product_details[attr] = int(''.join(filter(str.isdigit, text))) if any(
                                char.isdigit() for char in text) else 0
                    else:
                        product_details[attr] = detail_element.text
                except Exception:
                    product_details[attr] = 0

            # Extract dimensions
            dimensions_value = "0 x 0 x 0"
            for dim_attr in dimensions_attributes_cm:
                try:
                    dimension_element = WebDriverWait(driver, 1).until(
                        EC.presence_of_element_located((By.XPATH,
                                                        f'//div[@class="overview"]//span[@class="product-attr" and text()="{dim_attr}"]/following-sibling::span[@class="product-value"]'))
                    )
                    dimensions_value = dimension_element.text
                    break  # Exit loop if dimension is found
                except Exception:
                    continue

            # Split dimensions into Length, Breadth, and Height
            length = breadth = height = 0
            try:
                length_match = re.search(r"(\d+(\.\d+)?)\s*[lL]", dimensions_value)
                breadth_match = re.search(r"(\d+(\.\d+)?)\s*[wW]", dimensions_value)
                height_match = re.search(r"(\d+(\.\d+)?)\s*[hH]", dimensions_value)

                if length_match:
                    length = float(length_match.group(1))
                if breadth_match:
                    breadth = float(breadth_match.group(1))
                if height_match:
                    height = float(height_match.group(1))
            except Exception:
                pass

            product_details["Length (Cm)"] = length
            product_details["Breadth (Cm)"] = breadth
            product_details["Height (Cm)"] = height

            try:
                ratings_element = WebDriverWait(driver, 1).until(
                    EC.presence_of_element_located((By.XPATH,'//li[@class="custom-scroll-bottom" and @data-href="#reviewViewerDiv"]//span[@class="product-attr" and text()="Ratings"]/following-sibling::span[@class="product-value"]'))
                )
                product_details["Ratings"] = ratings_element.text.split()[0]
            except Exception:
                product_details["Ratings"] = 0

            # Adding sold count to product details
            product_details["Sold Count"] = sold_count

            print(f"Product Details: {product_details}")
        except Exception as e:
            print(f"Error retrieving product details: {e}")

        # Close the new tab
        driver.close()

        # Switch back to the main tab
        driver.switch_to.window(driver.window_handles[0])

        # Increment the product count
        product_count += 1
    except Exception as e:
        print(f"Error clicking the product image: {e}")

# Close the driver after all products have been processed
time.sleep(5)
driver.quit()
