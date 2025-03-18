clc; clear; clf; close all;

% Specify the folder containing reference images
ref_folder = uigetdir('', 'Select the Folder Containing Reference Images');

% Get a list of all image files in the folder
image_files = dir(fullfile(ref_folder, '.'));
image_files = image_files(~[image_files.isdir]);  % Exclude directories

% Initialize a cell array to hold properties of reference signatures
ref_props_array = cell(length(image_files), 1);

% Process each reference image
for i = 1:length(image_files)
    % Read the image
    ref_img = imread(fullfile(ref_folder, image_files(i).name));
    
    % Convert to Grayscale
    ref_gray = rgb2gray(ref_img);
    
    % Binarization using Otsu's thresholding
    ref_bw = imbinarize(ref_gray);
    
    % Invert if necessary (signature should be white on black background)
    ref_bw = ~ref_bw;
    
    % Remove Noise and Keep Largest Connected Component (Main Signature)
    ref_bw = bwareafilt(ref_bw, 1);
    
    % Extract Features
    ref_props_array{i} = regionprops(ref_bw, 'Area', 'Perimeter', 'Centroid');
end

% Load Test Signature Image
[filename, pathname] = uigetfile({'.';'.bmp';'.jpg';'*.gif';'.PNG'}, 'Pick a Test Signature Image File');
test_img = imread([pathname, filename]);

% Convert to Grayscale (test image)
test_gray = rgb2gray(test_img);

% Binarization using Otsu's thresholding (test image)
test_bw = imbinarize(test_gray);

% Invert if necessary (signature should be white on black background)
test_bw = ~test_bw;

% Remove Noise and Keep Largest Connected Component (Main Signature)
test_bw = bwareafilt(test_bw, 1);

% Extract properties for the test signature
test_props = regionprops(test_bw, 'Area', 'Perimeter', 'Centroid');

% Ensure a valid signature is detected in the test image
is_verified = false;
accuracy = 0;

if isempty(test_props)
    disp('Error: Could not extract features from test signature.');
else
    % Verification Loop
    threshold1 = 0.001; % Adjust based on accuracy needs
    threshold2 = 0.05;
    for i = 1:length(ref_props_array)
        if ~isempty(ref_props_array{i}) % Check if reference features are detected
            ref_area = ref_props_array{i}(1).Area;
            ref_perimeter = ref_props_array{i}(1).Perimeter;

            test_area = test_props(1).Area;
            test_perimeter = test_props(1).Perimeter;

            area_diff = abs(ref_area - test_area) / ref_area;
            perimeter_diff = abs(ref_perimeter - test_perimeter) / ref_perimeter;

            % Compute accuracy
            area_accuracy = (1 - area_diff) * 100;
            perimeter_accuracy = (1 - perimeter_diff) * 100;
            
            % Final accuracy as an average of the two
            accuracy = (area_accuracy + perimeter_accuracy) / 2;
            
            if area_diff < threshold1 && perimeter_diff < threshold2
                is_verified = true;
                break;  % Match found, no need to check further
            end
        end
    end
end

if is_verified
    disp('Signature Verified: MATCH');
    fprintf('Matching Accuracy: %.2f%%\n', accuracy);
else
    disp('Signature NOT Verified: MISMATCH');
    fprintf('Matching Accuracy: %.2f%%\n', accuracy);
end

% Display Results
figure;
subplot(1,2,1); imshow(cat(3, ref_bw)); title('Reference Signatures'); 
subplot(1,2,2); imshow(cat(3, test_bw)); title('Test Signature');PROJECT
