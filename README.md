<h1>Thiết lập DataPipeline cho dữ liệu lớn từ Cloud</h1>
Bài tập này sử dụng một số phần mềm như sau:

Airflow Server - máy ảo chạy Ubuntu Server để bạn có thể sử dụng được Airflow (bạn có thể tham khảo ở phần cài đặt Airflow).
Spark - Bạn cần cài Spark trên máy ảo Airflow Server để thực hiện các thao tác xử lý dữ liệu.
MongoDB - Bạn cần cài MongoDB trên máy ảo Airflow Server để thực hiện các thao tác lưu trữ dữ liệu.

<h2> Tài nguyên</h2>

Bạn có thể tải Dataset ở [link sau](https://drive.google.com/file/d/1ep8YnyZW49034r-DK5swELkjEj6mZQLu/view?usp=sharing).  
Tập dữ liệu này sẽ gồm 2 file có cấu trúc như sau:

1. Questions.csv:

File csv chứa các thông tin liên quan đến câu hỏi của hệ thống, với cấu trúc như sau:

- Id: Id của câu trả lời.

- OwnerUserId: Id của người tạo câu trả lời đó. (Nếu giá trị là NA thì tức là không có giá trị này).

- CreationDate: Ngày câu hỏi được tạo.

- ClosedDate: Ngày câu hỏi kết thúc (Nếu giá trị là NA thì tức là không có giá trị này).

- Score: Điểm số mà người tạo nhận được từ câu hỏi này.

- Title: Tiêu đề của câu hỏi.

- Body: Nội dung câu hỏi.

2. File Answers.csv

File csv chứa các thông tin liên quan đến câu trả lời và có cấu trúc như sau:

- Id: Id của câu hỏi.

- OwnerUserId: Id của người tạo câu hỏi đó. (Nếu giá trị là NA thì tức là không có giá trị này)

- CreationDate: Ngày câu trả lời được tạo.

- ParentId: ID của câu hỏi mà có câu trả lời này.

- Score: Điểm số mà người trả lời nhận được từ câu trả lời này.

- Body: Nội dung câu trả lời.

<h2> Yêu cầu chi tiết</h2>

<h3> 1.  Task: start và end </h3>

Bạn cần tạo 2 task start và end là các DummyOperator để thể hiện cho việc bắt đầu và kết thúc của DAG.

<h3> 2. Task: branching </h3>

Bạn cần tạo 1 task để kiểm tra xem hai file Questions.csv và Answers.csv đã được tải xuống để sẵn sàng import hay chưa.

Nếu file chưa được tải xuống thì bắt đầu quá trình xử lý dữ liệu (Chuyển đến task clear_file).
Nếu file đã được tải xuống thì sẽ kết thúc Pipeline (Chuyển đến task end).
Để làm được Task này. bạn sẽ cần sử dụng BranchPythonOperator.

<h3> 3. Task: clear_file </h3>

Đây sẽ là Task đầu tiên trong quá trình xử lý dữ liệu,  trước khi Download các file Questions.csv và Answers.csv thì bạn sẽ cần xóa các file đang tồn tại để tránh các lỗi liên quan đến việc ghi đè. Để hoàn thành thao tác này bạn có thể sử dụng BashOperator.

<h3> 4. Task: download_question_file_task và download_answer_file_task </h3>

Bạn sẽ cần tạo 1 Task để download các file csv cần thiết. Đầu tiên, bạn hãy upload các file csv đó lên Google Drive cá nhân, sau đó sử dụng thư viện google_drive_downloader  và PythonOperator để tải các file đó. Ví dụ như sau:


from google_drive_downloader import GoogleDriveDownloader as gdd

gdd.download_file_from_google_drive(file_id='1iytA1n2z4go3uVCwE__vIKouTKyIDjEq',
                                    dest_path='./data/mnist.zip',
                                    unzip=True)
Đoạn code trên sẽ tải file theo id và lưu xuống đường dẫn dest_path. Để lấy được ID của file trên Google Drive, bạn sẽ cần lấy shareable link của file đó, ví dụ với link như sau:

Lưu ý: Nếu như bạn sử dụng thư viện và bị lỗi về "Virus Scan", bạn có thể tải đoạn code sau đây và sử dụng để tải các file từ Google Drive.

<h3> 5. Task: import_questions_mongo và import_answers_mongo </h3>

Sau khi tải xuống các file dữ liệu ở dạng csv, bạn sẽ cần Import các dữ liệu đó vào MongoDB để lưu trữ dữ liệu. Bạn có thể sử dụng BashOperator với câu lệnh mongoimport như sau:

mongoimport --type csv -d <database> -c <collection> --headerline --drop <file>
<h3> 6. Task: spark_process </h3>

Bạn cần tạo một Task để có thể sử dụng Spark xử lý các dữ liệu vừa được Import, dữ liệu sẽ được xử lý theo yêu cầu như sau:

Dựa vào tập dữ liệu, hãy tính toán xem mỗi câu hỏi đang có bao nhiêu câu trả lời. Dữ liệu đầu ra sẽ có cấu trúc như sau:

![image](https://github.com/QSDE2607/Airflow-1-/assets/171625181/2bd53d40-5201-4897-aa5f-2861a18d2d0a)

Sau khi tính toán xong, hãy sử dụng DataFrameWriter để lưu dữ liệu đã được xử lý ở dưới dạng .csv

Để hoàn thành task này, bạn có thể sử dụng SparkSubmitOperator để có thể submit một Spark Job vào hệ thống.

<h3> 7. Task: import_output_mongo </h3>

Sau khi đã xử lý xong dữ liệu, bạn hãy lưu các kết quả vào MongoDB dựa vào file .csv đã được export từ Spark. Tương tự với các Task import, bạn có thể sử dụng BashOperator với câu lệnh mongoimport.

<h3> 8. Sắp xếp thứ tự các Task và thiết lập để thực thi song song: </h3>

Bạn sẽ cần sắp xếp lại thứ tự của các Task theo như hình dưới đây:

![image](https://github.com/QSDE2607/Airflow-1-/assets/171625181/57af556f-37c7-411e-aee7-2a00ff3d9a9a)


Đồng thời, cũng có những Task có thể được thực thi song song với nhau để tiết kiệm thời gian hơn:

dowload_question_file_task và dowload_answer_file_task
import_questions_mongo và import_answers_mongo
Bạn hãy thiết lập cho Airflow sử dụng LocalExecutor để có thể thực thi các Task song song.

<h3> 9. (Nâng cao) Cài đặt CeleryExecutor cho Airflow </h3>

Ngoài LocalExecutor thì Airflow cũng hỗ trợ sử dụng CeleryExecutor để thực thi các Task trên các Worker khác nhau. Bạn hãy cài đặt và thiết lập cho Airflow có thể sử dụng CeleryExecutor. Bạn có thể tham khảo Bài 16: Chạy Data Pipeline song song để có thể hoàn thành thao tác này.
