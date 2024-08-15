# TP-CTF_Writeup

- Trong khi vào stalk github của một người anh thì mình phát hiện được một chall khá hay về file upload của PHP.
- Sau vài ngày suffer với nó thì cuối cùng mình cũng đã solve được nó.
- Bài viết này sẽ là góc nhìn của mình về lỗ hỏng file upload cũng như polyglot file và serialized trong PHP.

```php
<?php
class Log
{
    function __destruct()
    {
        file_put_contents($this->filename, $this->data);
    }
}

if (isset($_GET['debug']))
    highlight_file(__FILE__);
else {
    if (isset(($_GET['file'])) && file_exists($_GET['file'])) {
        echo "<center><img src=\"" . $_GET['file'] . "\"></center>";
    }

    if (isset($_POST["submit"])) {
        if (!is_dir("uploads"))
            mkdir("uploads", 0777, true);
        $target_dir = "uploads/";
        $target_file = $target_dir . basename($_FILES["fileToUpload"]["name"]);
        $uploadOk = 1;
        $imageFileType = strtolower(pathinfo($target_file, PATHINFO_EXTENSION));
        // Check if image file is a actual image or fake image

        $check = getimagesize($_FILES["fileToUpload"]["tmp_name"]);
        if ($check !== false) {
            echo "File is an image - " . $check["mime"] . ".";
            $uploadOk = 1;
        } else {
            echo "File is not an image.";
            $uploadOk = 0;
        }

        // Check if file already exists
        if (file_exists($target_file)) {
            echo "Sorry, file already exists.";
            $uploadOk = 0;
        }
        // Check file size
        if ($_FILES["fileToUpload"]["size"] > 500000) {
            echo "Sorry, your file is too large.";
            $uploadOk = 0;
        }
        // Allow certain file formats
        if (
            $imageFileType != "jpg" && $imageFileType != "png" && $imageFileType != "jpeg"
            && $imageFileType != "gif"
        ) {
            echo "Sorry, only JPG, JPEG, PNG & GIF files are allowed.";
            $uploadOk = 0;
        }
        // Check if $uploadOk is set to 0 by an error
        if ($uploadOk == 0) {
            echo "Sorry, your file was not uploaded.";
            // if everything is ok, try to upload file
        } else {
            if (move_uploaded_file($_FILES["fileToUpload"]["tmp_name"], $target_file)) {
                echo "The file " . basename($_FILES["fileToUpload"]["name"]) . " has been uploaded.";
            } else {
                echo "Sorry, there was an error uploading your file.";
            }
        }
    }
?>

<form
    action="index.php"
    method="post"
    enctype="multipart/form-data"
>
    Select image to upload
    <input
        type="file"
        name="fileToUpload"
        id="fileToUpload"
    >
    <input
        type="submit"
        value="Upload Image"
        name="submit"
    >
</form>
<!-- /?debug -->
<?php } ?>
```

- Trong lần đầu đọc src cũng như giải thử, mình nghĩ nó sẽ không thể solve được vì các fillter mà author đặt ra.
- Việc whitelist file extention cộng với các kỹ thuật xử lý đường dẫn file chặt chẽ dẫn đến chuyện mình up một file shell của PHP để thực thi lệnh gần như là no hope.
- Vài hôm sau mình nhận ra, phần src có vài điểm khá là lạ, một số chỗ hình như bị "thừa".
- Nếu để ý, chúng ta sẽ thấy, Class Log được định nghĩa nhưng chưa từng được khởi tạo và sử dụng trong đoạn code.
```php

class Log
{
    function __destruct()
    {
        file_put_contents($this->filename, $this->data);
    }
}
```

- Nhìn qua thì Class này chứa một magic method khá sus là ```__destruct()``` tại đây nó có thể thực hiện ghi data vào một file.
- Tới đây thì mội thứ bắt đầu quay trở lại một vòng lặp.
- Nếu Upload một đoạn PHP để chỉnh sửa cái class hoặc sử dụng nó thì ta có thể ghi thêm một file mới, từ đó có thể RCE. TUY NHIÊN nếu đã upload được một đoạn PHP thì tội gì chúng ta không up thẳng một con shell cơ mà. Tội gì phải ghi thêm file mới. Vậy thì cái Class đó để làm gì.

![image](https://github.com/user-attachments/assets/e6e63175-fd57-418e-9255-c9c6a0c458a2)

- Tới đây thì mình nghĩ tới một hướng, đó là kết hợp cả File Upload và Deserialize 

