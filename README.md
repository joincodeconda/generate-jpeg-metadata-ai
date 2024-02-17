# generate-jpeg-metadata-ai

AI-generated imagery is becoming increasingly prevalent, and photographers and digital artists are constantly seeking ways to add more depth and context to their creations. This tutorial will walk you through a comprehensive process to enrich your AI-generated photos with contextual metadata using [PhotoTag.ai's API](http://www.phototag.ai/api), Python, and a user-friendly desktop application built with PyQt5.

## Step 1: Preparing Your Assets and Environment

### Creating an API Token on PhotoTag.ai
1. Visit the [API page on PhotoTag.ai](http://www.phototag.ai/api).
2. If you haven't already, sign up or log in to access the API section.
3. Navigate to the API tokens area and create a new token. Note this token down securely; you'll need it for your script.

### Where to Save Your Photos
- Organize your AI-generated photos within a specific folder on your computer. Ideally, choose a location that's easily accessible, like `D:\AI_Photos` or `~/AI_Photos` on Mac.

### Preparing the Python Script
1. Ensure Python is installed on your system. If not, download it from the [official Python website](https://www.python.org/downloads/).
2. Install the required Python packages (`requests`, `piexif`, `PyQt5`) by running the following command in your terminal or command prompt:
   ```bash
   pip install requests piexif PyQt5
   ```
### Saving the Provided Python Script

Copy the entire Python script provided below and save it to a known location on your computer. For convenience, you might save it in the same folder as your photos or in a dedicated scripts directory. This will make it easier to run the script and process your AI-generated photos for metadata enhancement.

```python
# Import necessary libraries for the script
import os  # For interacting with the operating system
import shutil  # For file operations like moving files
import sys  # Access to some variables used or maintained by the interpreter
import requests  # To make HTTP requests to a specified URL
import piexif  # To manipulate EXIF data within images
from PyQt5.QtWidgets import QApplication, QWidget, QPushButton, QVBoxLayout, QProgressBar, QLabel, QFileDialog, QTextEdit
from PyQt5.QtCore import Qt  # For creating the graphical user interface (GUI)

# Variable to store your API token for authentication
api_token = "" # Replace with your API token

def get_image_metadata(image_path, custom_context):
    """
    Fetches metadata for an image using the PhotoTag.ai API.

    :param image_path: The path to the image file.
    :param custom_context: A string of custom context to improve API results.
    """
    # The URL of the API endpoint
    url = "https://server.phototag.ai/api/keywords"
    # Headers for the request, including the authorization token
    headers = {"Authorization": f"Bearer {api_token}"}
    # The payload of the request, including language, maximum keywords, and custom context
    payload = {
        "language": "en",
        "maxKeywords": 40,
        "customContext": custom_context
    }
    # Open the image file in binary mode and send the request
    with open(image_path, 'rb') as img_file:
        files = {"file": img_file}
        response = requests.post(url, headers=headers, data=payload, files=files)

    # If the request is successful (status code 200), process the data
    if response.status_code == 200:
        data = response.json().get("data")
        if data:
            # Extract the title, description, and keywords from the response
            title = data.get("title", "")
            description = data.get("description", "")
            keywords = data.get("keywords", [])
            # Log the fetched metadata
            print(f"Metadata fetched for image: {image_path}")
            print(f"Title: {title}")
            print(f"Description: {description}")
            print(f"Keywords: {keywords}")
            return title, description, keywords
    else:
        # Log failure if the request was unsuccessful
        print(f"Failed to fetch metadata. Check your API token. Status code: {response.status_code}")
    return None, None, []

def write_metadata_to_image(image_path, title, description, keywords):
    """
    Writes fetched metadata back into the image's EXIF data.

    :param image_path: The path to the image file.
    :param title: The title to write into the image's metadata.
    :param description: The description to write into the image's metadata.
    :param keywords: A list of keywords to write into the image's metadata.
    """
    try:
        # Load the existing EXIF data from the image
        exif_dict = piexif.load(image_path)
        # Prepare the metadata for insertion
        keywords_str = ', '.join(keywords)
        # Encode the metadata in appropriate formats
        title_bytes = title.encode('utf-16le')
        description_bytes = description.encode('utf-8')
        keywords_bytes = keywords_str.encode('utf-16le')
        # Update the EXIF data dictionary with new metadata
        exif_dict['0th'][piexif.ImageIFD.ImageDescription] = description_bytes
        exif_dict['0th'][piexif.ImageIFD.XPTitle] = title_bytes
        exif_dict['0th'][piexif.ImageIFD.XPKeywords] = keywords_bytes
        # Dump the updated EXIF data back into the image
        exif_bytes = piexif.dump(exif_dict)
        piexif.insert(exif_bytes, image_path)
        # Log success
        print(f"Metadata successfully written to the image: {image_path}")
        return True
    except Exception as e:
        # Log any errors that occur
        print(f"Failed to write metadata to the image: {str(e)}")
        return False

class ImageKeywordingTool(QWidget):
    """
    A graphical user interface (GUI) tool for processing a folder of images,
    fetching metadata for each, and writing it back to the images.
    """
    def __init__(self):
        super().__init__()
        self.initUI()

    def initUI(self):
        # Set up the window title and size
        self.setWindowTitle('Image Keywording Tool')
        self.resize(600, 400)
        layout = QVBoxLayout()

        # Create UI elements: a text box for status messages, a button to select folders, and a progress bar
        self.status_message = QTextEdit()
        self.status_message.setPlainText("Processing Not Started")
        self.status_message.setReadOnly(True)
        self.select_folder_button = QPushButton('Select Folder')
        self.select_folder_button.clicked.connect(self.start_processing)
        self.progress_bar = QProgressBar()

        # Add the UI elements to the layout
        layout.addWidget(self.status_message)
        layout.addWidget(self.select_folder_button)
        layout.addWidget(self.progress_bar)
        self.setLayout(layout)

    def start_processing(self):
        # Function to handle the folder selection and start processing images
        selected_folder = QFileDialog.getExistingDirectory(self, "Select Folder")
        if selected_folder:
            self.status_message.setPlainText("Processing Running...")
            self.process_images_in_folder(selected_folder)
            self.status_message.append("Processing Completed")
            self.status_message.append("Close Window to Exit")

    def process_images_in_folder(self, folder_path):
        # Set up 'ready' and 'failed' folders for processed images
        ready_folder = os.path.join(folder_path, "ready")
        failed_folder = os.path.join(folder_path, "failed")
        os.makedirs(ready_folder, exist_ok=True)
        os.makedirs(failed_folder, exist_ok=True)

        # Identify all JPEG images in the folder
        images_to_process = [filename for filename in os.listdir(folder_path) if filename.lower().endswith(('.jpg', '.jpeg'))]
        total_images = len(images_to_process)
        processed_images = 0

        # Process each image, updating its metadata
        for filename in images_to_process:
            image_path = os.path.join(folder_path, filename)
            # Use parts of the filename as custom context for the API request
            custom_context = ' '.join([c for c in filename.split('.')[0].split('_') if c != 'g' and not c.isdigit()])
            title, description, keywords = get_image_metadata(image_path, custom_context)
            if title and keywords:
                success = write_metadata_to_image(image_path, title, description, keywords)
                if success:
                    shutil.move(image_path, os.path.join(ready_folder, filename))
                else:
                    shutil.move(image_path, os.path.join(failed_folder, filename))
            else:
                shutil.move(image_path, os.path.join(failed_folder, filename))

            # Update the progress bar based on the number of processed images
            processed_images += 1
            progress = processed_images / total_images * 100
            self.progress_bar.setValue(int(round(progress)))
            QApplication.processEvents()  # Ensure the GUI remains responsive

def main():
    # Initialize and run the application
    app = QApplication(sys.argv)
    ex = ImageKeywordingTool()
    ex.show()
    sys.exit(app.exec_())

if __name__ == '__main__':
    main()
```

Ensure you replace `api_token = ""` with your actual API token from PhotoTag.ai. This script is designed to enhance AI-generated photos by embedding fetched metadata directly into your image files. It's a powerful way to add context and make your digital art more searchable and organized.

## Step 2: Integrating the API Token and Running the Script

### Inserting Your API Token
- Open the Python script with a text editor or IDE.
- Locate the line `api_token = ""` and insert your API token between the quotes.

### Running the Script
1. Open a terminal or command prompt.
2. Navigate to the directory where you saved the Python script.
3. Run the script by typing:
   ```bash
   python name_of_your_script.py
   ```
4. The graphical user interface (GUI) of the Image Keywording Tool will launch, indicating that the script is ready to run.

## Step 3: Using the Image Keywording Tool

### Selecting Your Photo Folder
1. Click on the 'Select Folder' button in the tool's GUI.
2. Navigate to and select the folder containing your AI-generated photos.
3. The tool will begin processing each photo, fetching metadata from PhotoTag.ai based on the file names, and applying this metadata to the photos.

### Understanding the Process
- The script parses each file name of your AI-generated photos to extract additional context, which it then sends to PhotoTag.ai to generate relevant metadata (titles, descriptions, keywords).
- Successfully processed photos will be moved to a `ready` subfolder, while any that fail (due to errors or inability to fetch metadata) will be moved to a `failed` subfolder.

## Key Points to Remember
- This tutorial and script are specifically designed for AI-generated photos, leveraging the unique aspects of their file names to add contextual metadata.
- Ensure your API token is kept secure and is correctly inserted into the script before running it.
- The PyQt5 GUI makes it easy to process multiple photos at once, providing a progress bar and status updates throughout the process.

## Conclusion
By following this tutorial, you can add a new layer of depth to your AI-generated photos, making them more searchable, relatable, and valuable. Whether you're a photographer, digital artist, or hobbyist, this approach offers a straightforward way to enhance your digital creations with contextual metadata, all from the comfort of your desktop.

Remember, the key to a successful integration lies in the preparation of your assets, the secure handling of your API token, and the careful organization of your photos for processing. Happy tagging!
