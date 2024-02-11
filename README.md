# epub_ingestion_python

e.g. Use epub books from Humble Bunbles in Retrieval Augmented Generation systems with Foundation Models.

```python
#########################################################
# This code automaticaly finds and processes epub books
# esspecially for RAG document ingestion processing
#########################################################
"""
# Set of Results:
1. One .jsonl file
2. Individual .json files in a folder
3. One .txt text file of the whole epub
4. Individual .txt files from separate parts of epub
5. chunks under specified character length as separate text files
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


def extract_text_from_html(html_content):
    """
    Extracts and returns text from an HTML content.
    """
    #print("HTML Content before BeautifulSoup Parsing:\n", html_content[:500])  # Print first 500 characters of HTML
    print(f"\nlen(HTML Content before BeautifulSoup Parsing) -> {len(html_content)}")  # Print first 500 characters of HTML

    soup = BeautifulSoup(html_content, 'html.parser')
    parsed_text = soup.get_text()
    # print("Extracted Text:\n", parsed_text[:500])  # Print first 500 characters of extracted text
    print(f"len(Extracted Text) -> {len(parsed_text)}")  # Print first 500 characters of extracted text

    return parsed_text

def fix_text_formatting(text):
    """Replaces the Unicode right single quotation mark with a standard apostrophe."""
    return text.replace("\u2019", "'")


def save_individual_chunks(chunks_list, output_chunks_dir, chunk_source_name):

      for index, this_chunk in enumerate(chunks_list):
          chunk_name = f"{chunk_source_name}_{index}.txt"

          # Save individual txt files
          individual_chunk_path = os.path.join(output_chunks_dir, chunk_name)
          with open(individual_chunk_path, 'w') as f:
              f.write(this_chunk)
              # print('chunk writen', individual_chunk_path)

      return len(chunks_list)


def append_chunks_to_jsonl(chunks_list, output_chunks_jsonl_path, chunk_source_name):
    """Appends chunks of text to a .jsonl file, each chunk as a JSON object.

    Args:
        chunks_list (list): List of text chunks to be appended.
        output_jsonl_path (str): The output file path for the .jsonl file.
        chunk_source_name (str): Base name for each chunk, used in the 'source_name' field.

    Returns:
        int: The number of chunks appended.
    """

    with open(output_chunks_jsonl_path, 'a') as f:  # Open file in append mode
        for index, this_chunk in enumerate(chunks_list):
            # Construct a JSON object for the chunk
            chunk_data = {
                "source_name": f"{chunk_source_name}_{index}",
                "text": this_chunk
            }
            
            # Convert the chunk data to a JSON string and append it to the file with a newline
            f.write(json.dumps(chunk_data) + '\n')

    return len(chunks_list)


def extract_text_from_epub(epub_file_path, output_jsonl_path, output_json_dir, output_whole_txt_path, output_txt_dir, output_chunks_jsonl_path, output_chunks_dir, chunk_size=500):
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

        # Create a directory for chunks output_chunks_dir
        if not os.path.exists(output_chunks_dir):
            os.makedirs(output_chunks_dir)

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


                #######
                # json
                #######

                # Write/Append to a single JSONL file
                with open(output_jsonl_path, 'a') as f:
                    json_record = json.dumps({'text': text.strip()})
                    f.write(json_record + '\n')

                # Save individual JSON file
                individual_json_path = os.path.join(output_json_dir, f"{os.path.splitext(html_file)[0]}.json")
                with open(individual_json_path, 'w') as f:
                    json.dump({'text': text.strip()}, f, indent=4)

                #######
                # txt
                #######

                # Write/Append to a single text .txt file
                with open(output_whole_txt_path, 'a') as f:
                    f.write(text + '\n\n')

                # Save individual txt files
                individual_txt_path = os.path.join(output_txt_dir, f"{os.path.splitext(html_file)[0]}.txt")
                with open(individual_txt_path, 'w') as f:
                    f.write(text)

                #########
                # Chunks
                #########

                chunks_list = make_chunk_list(text, chunk_size)

                chunk_source_name = os.path.splitext(html_file)[0]

                number_of_chunks = save_individual_chunks(chunks_list, output_chunks_dir, chunk_source_name)
                print(f"Chunked: split into this many chunks-> {number_of_chunks}")

                append_chunks_to_jsonl(chunks_list, output_chunks_jsonl_path, chunk_source_name)

                print(f"{html_file} -> ok!")

            else:
                print(f"Warning: File {full_path} not found in the archive.")


def zip_folder(path_to_directory_to_zip, output_destination_zip_file_path):
    """Creates a zip archive of a specified folder.

    Args:
        path_to_directory_to_zip (str): The path to the folder to be zipped.
        output_destination_zip_file_path (str): The desired name and path of the output zip file.
    """
    # # Specify the folder you want to zip
    # path_to_directory_to_zip = "individual_jsons"

    # # Specify the desired output zip file name (e.g., 'jsons_archive.zip')
    # output_destination_zip_file_path = "jsons_archive_zip"

    shutil.make_archive(output_destination_zip_file_path, 'zip', path_to_directory_to_zip)


###########
# Chunking
############

import re

def split_sentences_and_punctuation(text):
    """Splits text into sentences, attempting to preserve punctuation and all text content.

    Args:
        text (str): The input text.

    Returns:
        list: A list of sentences with preserved punctuation.
    """

    # This pattern attempts to split at sentence endings (.?!), including the punctuation with the preceding sentence
    # It uses a lookahead to keep the punctuation with the sentence
    sentence_end_regex = r'(?<=[.!?])\s+(?=[A-Z])'

    split_sentences_and_punctuation_list = re.split(sentence_end_regex, text)

    # Optionally, remove empty strings if they are not desired
    split_sentences_and_punctuation_list = [s for s in split_sentences_and_punctuation_list if s]

    print("split_sentences_and_punctuation_list")
    print(split_sentences_and_punctuation_list)

    return split_sentences_and_punctuation_list


def recombine_punctuation(sentences):
    """
    A helper function a that
    Recombines floating punctuation and
    creates a new list of sentences."""
    recombined_sentences = []
    i = 0

    while i < len(sentences) - 1:
        # print(i)
        # print(sentences[i-1])
        # print(sentences[i])
        # print(sentences[i+1])

        sentence = sentences[i].strip()
        next_item = sentences[i + 1].strip()

        # if next_item in ".?!":
        if re.match(r"[.?!]+", next_item):
            recombined_sentences.append(sentence + next_item)
            i += 1  # Skip the punctuation since it's been combined
        else:
            recombined_sentences.append(sentence)
        i += 1

    # Add the last sentence (if it exists)
    if sentences[-1]:
        recombined_sentences.append(sentences[-1].strip())

    return recombined_sentences


def chunk_text(sentences, chunk_size):
    chunked_text = []
    current_chunk = ""

    for sentence in sentences:
        # Case 1: Chunk + sentence easily fit
        if len(current_chunk) + len(sentence) + 1 <= chunk_size:
            current_chunk += sentence + " "

        # Case 2: Sentence itself is too big
        elif len(sentence) > chunk_size:
            # Split long sentence (implement 'split_long_sentence' below)
            for sub_sentence in split_long_sentence(sentence, chunk_size):
                chunked_text.append(sub_sentence.strip())

        # Case 3:  Chunk + sentence exceed limit, time to split
        else:
            chunked_text.append(current_chunk.strip())
            current_chunk = sentence + " "

    # Handle final chunk
    if current_chunk:
        chunked_text.append(current_chunk.strip())

    return chunked_text


def split_long_sentence(sentence, chunk_size):
    """Splits a long sentence into chunks, aiming near the  chunk_size."""
    words = sentence.split()
    chunks = []
    current_chunk = ""

    for word in words:
        if len(current_chunk) + len(word) + 1 <= chunk_size:
            current_chunk += word + " "
        else:
            chunks.append(current_chunk.strip())
            current_chunk = word + " "

    if current_chunk:
        chunks.append(current_chunk.strip())

    return chunks


def check_for_not(chunk, window_size=25):
    """Checks if 'not' is isolated near a potential cut.

    Args:
        chunk (list): A list of sentences forming the chunk.
        window_size (int): The number of characters to consider on either side.

    Returns:
        bool: True if 'not' is isolated, False otherwise.
    """

    joined_chunk = ' '.join(chunk)
    not_indices = [m.start() for m in re.finditer(r'\bnot\b', joined_chunk)]

    for index in not_indices:
        start = max(0, index - window_size)
        end = min(len(joined_chunk), index + window_size)
        if not re.search(r'\w', joined_chunk[start:end]):  # Check for surrounding words
            return True

    return False


def make_chunk_list(text, chunk_size):
    split_sentences_list = split_sentences_and_punctuation(text)
    chunk_list = chunk_text(split_sentences_list, chunk_size)

    for i in chunk_list:
        if not i:
            print("error None in chunk_list: make_chunk_list()")

    print("len chunk list", len(chunk_list))

    return chunk_list


################
# Example usage
################
"""
1. have your source epub in the cwd and 
2. change the value of epub_file_path to the name of your file
"""
epub_file_path = 'rustforrustaceans.epub' # Replace with your EPUB file path

# json
output_jsonl_path = 'output.jsonl'
output_json_dir = 'individual_jsons' # Directory to store individual JSON files

# txt
output_whole_txt_path = 'whole.txt'
output_txt_dir = 'individual_txt' # Directory to store individual txt files

# chunks
output_chunks_dir = 'chunk_text_files' # Directory to store individual txt files

output_chunks_jsonl_path = 'chunks_jsonl_all' # Directory to store individual txt files

# run
extract_text_from_epub(epub_file_path, output_jsonl_path, output_json_dir, output_whole_txt_path, output_txt_dir, output_chunks_jsonl_path, output_chunks_dir, chunk_size=500)

# Call the zip function
zip_folder(output_json_dir, 'jsons_zip_archive')
zip_folder(output_txt_dir, 'txt_zip_archive')
zip_folder(output_chunks_dir, 'chunks_zip_archive')


```
