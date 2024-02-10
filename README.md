# epub_ingestion_python

e.g. Use epub books from Humble Bunbles in Retrieval Augmented Generation systems with Foundation Models.

```python
#########################################################
# This code automaticaly finds and processes epub books
# esspecially for RAG document ingestion processing
#########################################################
"""
# Set of Results:
1. One jsonl file
2. Individual json files in a folder
3. One .txt text file of the whole epub
4. Individual txt files from separate parts of epub
"""

import zipfile
import xml.etree.ElementTree as ET
from bs4 import BeautifulSoup
import json
import os
import shutil


def get_ordered_html_files(opf_content):
    """
    Parses the content.opf file to determine the reading order of HTML files in the EPUB.

    The function reads the 'content.opf' file, which contains metadata about the EPUB's structure.
    It identifies the 'spine' element, which lists the reading order of the content documents,
    and the 'manifest' element, which provides the location of these documents.
    The function returns a list of HTML file paths in the order they should be read.

    Args:
    opf_content (str): A string representation of the content.opf file.

    Returns:
    list: An ordered list of HTML file paths as specified in the EPUB's spine.
    """

    # Parse the content.opf XML content
    tree = ET.ElementTree(ET.fromstring(opf_content))
    root = tree.getroot()

    # Define the namespace for the OPF package file
    ns = {'opf': 'http://www.idpf.org/2007/opf'}

    # Find the spine element which indicates the order of the content documents
    spine = root.find('opf:spine', ns)
    itemrefs = spine.findall('opf:itemref', ns)

    # Extract the id references for each item in the spine
    item_ids = [itemref.get('idref') for itemref in itemrefs]

    # Find the manifest element which lists all the content documents
    manifest = root.find('opf:manifest', ns)
    items = manifest.findall('opf:item', ns)

    # Create a dictionary mapping item IDs to their corresponding file paths
    html_files = {item.get('id'): item.get('href') for item in items if item.get('media-type') == 'application/xhtml+xml'}

    # Generate an ordered list of HTML files based on the spine order
    ordered_html_files = [html_files[item_id] for item_id in item_ids if item_id in html_files]

    return ordered_html_files


# def extract_text_from_html(html_content):
#     """
#     Extracts and returns text from an HTML content.
#     """
#     soup = BeautifulSoup(html_content, 'html.parser')
#     return soup.get_text()

def extract_text_from_html(html_content):
    """
    Extracts and returns text from an HTML content.
    """
    #print("HTML Content before BeautifulSoup Parsing:\n", html_content[:500])  # Print first 500 characters of HTML
    print(f"\nlen(HTML Content before BeautifulSoup Parsing) -> {len(html_content)}")  # Print first 500 characters of HTML

    soup = BeautifulSoup(html_content, 'html.parser')
    parsed_text = soup.get_text()
    # print("Extracted Text:\n", parsed_text[:500])  # Print first 500 characters of extracted text
    print(f"\nLen(Extracted Text) -> {len(parsed_text)}")  # Print first 500 characters of extracted text

    return parsed_text

def fix_text_formatting(text):
    """Replaces the Unicode right single quotation mark with a standard apostrophe."""
    return text.replace("\u2019", "'")


def extract_text_from_epub(epub_file_path, output_jsonl_path, output_json_dir, output_whole_txt_path, output_txt_dir):
    with zipfile.ZipFile(epub_file_path, 'r') as epub:
        print("EPUB Contents:", epub.namelist())

        opf_file = [f for f in epub.namelist() if 'content.opf' in f][0]
        opf_content = epub.read(opf_file).decode('utf-8')

        ordered_html_files = get_ordered_html_files(opf_content)

        # Create a directory for individual JSON files
        if not os.path.exists(output_json_dir):
            os.makedirs(output_json_dir)

        # Create a directory for individual txt files
        if not os.path.exists(output_txt_dir):
            os.makedirs(output_txt_dir)

        # Read and extract text from each HTML file
        for html_file in ordered_html_files:
            full_path = os.path.join(os.path.dirname(opf_file), html_file)
            if full_path in epub.namelist():
                html_content = epub.read(full_path).decode('utf-8')

                #########################
                # extract text from epub
                #########################
                raw_text = extract_text_from_html(html_content)
                print(f"len(text for json)-> {len(raw_text)}")
                
                # fix text formatting
                text = fix_text_formatting(raw_text)

                # Write/Append to a single JSONL file
                with open(output_jsonl_path, 'a') as f:
                    json_record = json.dumps({'text': text.strip()})
                    f.write(json_record + '\n')

                # Save individual JSON file
                individual_json_path = os.path.join(output_json_dir, f"{os.path.splitext(html_file)[0]}.json")
                with open(individual_json_path, 'w') as f:
                    json.dump({'text': text.strip()}, f, indent=4)

                # Write/Append to a single text .txt file
                with open(output_whole_txt_path, 'a') as f:
                    f.write(text + '\n\n')

                # Save individual txt files
                individual_txt_path = os.path.join(output_txt_dir, f"{os.path.splitext(html_file)[0]}.txt")
                with open(individual_txt_path, 'w') as f:
                    f.write(text)

                print(f"{html_file} -> ok!")

            else:
                print(f"Warning: File {full_path} not found in the archive.")


def zip_folder(path_to_directory_to_zip='individual_jsons', output_destination_zip_file_path='jsons_archive_zip'):
    """Creates a zip archive of a specified folder.

    Args:
        path_to_directory_to_zip (str): The path to the folder to be zipped.
        output_destination_zip_file_path (str): The desired name and path of the output zip file.
    """
    # Specify the folder you want to zip
    path_to_directory_to_zip = "individual_jsons"

    # Specify the desired output zip file name (e.g., 'jsons_archive.zip')
    output_destination_zip_file_path = "jsons_archive_zip" 

    shutil.make_archive(output_destination_zip_file_path, 'zip', path_to_directory_to_zip)


################
# Example usage
################
epub_file_path = 'rustforrustaceans.epub' # Replace with your EPUB file path

# json
output_jsonl_path = 'output.jsonl'
output_json_dir = 'individual_jsons' # Directory to store individual JSON files

# txt
output_whole_txt_path = 'whole.txt'
output_txt_dir = 'individual_txt' # Directory to store individual txt files

# run 
extract_text_from_epub(epub_file_path, output_jsonl_path, output_json_dir, output_whole_txt_path, output_txt_dir)

# Call the zip function
zip_folder(output_json_dir, 'jsons_archive_zip')
zip_folder(output_txt_dir, 'txt_archive_zip')

```
