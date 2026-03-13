**Bài học Ngôn ngữ Assembly từ FFmpeg số Một**

**Giới thiệu**

Chào mừng bạn đến với Trường học Assembly của FFmpeg. Bạn đã thực hiện bước đầu tiên trong hành trình thú vị, đầy thử thách và cũng rất đáng giá trong lập trình. Những bài học này sẽ cung cấp cho bạn nền tảng về cách ngôn ngữ assembly được viết trong FFmpeg và giúp bạn hiểu rõ hơn những gì thực sự đang diễn ra bên trong máy tính của mình.

**Kiến thức yêu cầu**

* Kiến thức về C, đặc biệt là con trỏ. Nếu bạn chưa biết C, hãy học qua cuốn sách [The C Programming Language](https://en.wikipedia.org/wiki/The_C_Programming_Language)
* Toán học trung học (vô hướng vs vectơ, cộng, nhân, v.v.)

**Ngôn ngữ assembly là gì?**

Ngôn ngữ assembly là một ngôn ngữ lập trình nơi bạn viết mã tương ứng trực tiếp với các lệnh mà CPU xử lý. Assembly ở dạng con người có thể đọc được, đúng như tên gọi của nó, sẽ được *lắp ráp* (assembled) thành dữ liệu nhị phân, gọi là *mã máy* (machine code), mà CPU có thể hiểu. Bạn cũng có thể thấy mã assembly được gọi là “assembly” hoặc viết tắt là “asm”.

Phần lớn mã assembly trong FFmpeg thuộc loại gọi là *SIMD, Single Instruction Multiple Data*. SIMD đôi khi còn được gọi là lập trình vectơ. Điều này có nghĩa là một lệnh cụ thể sẽ hoạt động trên nhiều phần tử dữ liệu cùng lúc. Hầu hết các ngôn ngữ lập trình chỉ xử lý một phần tử dữ liệu tại một thời điểm, được gọi là lập trình vô hướng (scalar programming).

Như bạn có thể đoán, SIMD rất phù hợp để xử lý hình ảnh, video và âm thanh vì chúng chứa nhiều dữ liệu được sắp xếp tuần tự trong bộ nhớ. CPU có các lệnh chuyên dụng giúp xử lý loại dữ liệu tuần tự này.

Trong FFmpeg, bạn sẽ thấy các thuật ngữ “assembly function”, “SIMD”, và “vector(ise)” được dùng thay thế cho nhau. Tất cả đều nói đến cùng một việc: viết một hàm bằng ngôn ngữ assembly thủ công để xử lý nhiều phần tử dữ liệu cùng lúc. Một số dự án cũng gọi chúng là “assembly kernels”.

Tất cả những điều này có thể nghe khá phức tạp, nhưng điều quan trọng cần nhớ là trong FFmpeg, học sinh trung học cũng đã viết mã assembly. Cũng như mọi thứ khác, việc học bao gồm 50% là thuật ngữ chuyên môn và 50% là kiến thức thực sự.

**Tại sao chúng ta viết bằng ngôn ngữ assembly?**
Để việc xử lý đa phương tiện nhanh hơn. Việc đạt được cải thiện tốc độ 10 lần hoặc hơn khi viết mã assembly là rất phổ biến, điều này đặc biệt quan trọng khi muốn phát video theo thời gian thực mà không bị giật. Nó cũng tiết kiệm năng lượng và kéo dài thời lượng pin. Cũng cần lưu ý rằng các hàm mã hóa và giải mã video là một trong những hàm được sử dụng nhiều nhất trên thế giới, cả bởi người dùng cuối lẫn các công ty lớn trong trung tâm dữ liệu của họ. Vì vậy ngay cả một cải thiện nhỏ cũng nhanh chóng trở nên đáng kể.

Bạn thường sẽ thấy trên mạng mọi người sử dụng *intrinsics*, tức là các hàm giống C ánh xạ tới các lệnh assembly để cho phép phát triển nhanh hơn. Trong FFmpeg chúng tôi không dùng intrinsics mà viết mã assembly thủ công. Đây là một chủ đề gây tranh cãi, nhưng intrinsics thường chậm hơn khoảng 10–15% so với assembly viết tay (những người ủng hộ intrinsics sẽ không đồng ý), tùy thuộc vào trình biên dịch. Đối với FFmpeg, mỗi chút hiệu năng bổ sung đều quan trọng, vì vậy chúng tôi viết trực tiếp bằng assembly. Ngoài ra còn có lập luận rằng intrinsics khó đọc do sử dụng “[Hungarian Notation](https://en.wikipedia.org/wiki/Hungarian_notation)”.

Bạn cũng có thể thấy *inline assembly* (tức là không dùng intrinsics) vẫn còn tồn tại ở một vài nơi trong FFmpeg vì lý do lịch sử, hoặc trong các dự án như Linux Kernel vì những trường hợp sử dụng rất cụ thể. Đây là khi mã assembly không nằm trong một tệp riêng mà được viết trực tiếp cùng với mã C. Quan điểm phổ biến trong các dự án như FFmpeg là kiểu mã này khó đọc, không được nhiều trình biên dịch hỗ trợ và khó bảo trì.

Cuối cùng, bạn sẽ thấy nhiều “chuyên gia” tự xưng trên mạng nói rằng tất cả những điều này không cần thiết và trình biên dịch có thể tự thực hiện “vectorisation” cho bạn. Ít nhất cho mục đích học tập, hãy bỏ qua họ: các thử nghiệm gần đây, ví dụ trong [dự án dav1d](https://www.videolan.org/projects/dav1d.html), cho thấy vector hóa tự động chỉ đạt khoảng tăng tốc 2 lần, trong khi các phiên bản viết tay có thể đạt tới 8 lần.

**Các biến thể của ngôn ngữ assembly**
Các bài học này sẽ tập trung vào assembly x86 64-bit. Nó còn được gọi là amd64, mặc dù vẫn chạy trên CPU Intel. Ngoài ra còn có các loại assembly khác dành cho CPU khác như ARM và RISC-V, và có thể trong tương lai các bài học này sẽ được mở rộng để bao gồm chúng.

Có hai kiểu cú pháp assembly x86 mà bạn sẽ thấy trên mạng: AT&T và Intel. Cú pháp AT&T cũ hơn và khó đọc hơn so với cú pháp Intel. Vì vậy chúng ta sẽ sử dụng cú pháp Intel.

**Tài liệu hỗ trợ**
Bạn có thể ngạc nhiên khi biết rằng sách hoặc tài nguyên trực tuyến như Stack Overflow không thực sự hữu ích làm tài liệu tham khảo. Một phần là vì chúng ta chọn sử dụng assembly viết tay với cú pháp Intel. Ngoài ra, nhiều tài nguyên trực tuyến tập trung vào lập trình hệ điều hành hoặc phần cứng, thường sử dụng mã không SIMD. Assembly trong FFmpeg đặc biệt tập trung vào xử lý hình ảnh hiệu năng cao, và như bạn sẽ thấy, đây là một cách tiếp cận khá độc đáo đối với lập trình assembly. Tuy vậy, sau khi hoàn thành các bài học này, bạn sẽ dễ dàng hiểu các trường hợp sử dụng assembly khác.

Nhiều cuốn sách đi sâu vào chi tiết kiến trúc máy tính trước khi dạy assembly. Điều này ổn nếu đó là điều bạn muốn học, nhưng từ góc nhìn của chúng tôi, điều đó giống như học về động cơ trước khi học lái xe.

Tuy nhiên, các sơ đồ ở phần sau của cuốn “The Art of 64-bit assembly” minh họa các lệnh SIMD và hành vi của chúng dưới dạng trực quan là khá hữu ích:
https://artofasm.randallhyde.com/

Một server Discord có sẵn để trả lời câu hỏi:
https://discord.com/invite/Ks5MhUhqfB

**Thanh ghi (Registers)**
Thanh ghi là các vùng trong CPU nơi dữ liệu có thể được xử lý. CPU không thao tác trực tiếp trên bộ nhớ; thay vào đó dữ liệu được nạp vào thanh ghi, được xử lý, rồi ghi trở lại bộ nhớ. Trong assembly, nói chung bạn không thể sao chép trực tiếp dữ liệu từ một vị trí bộ nhớ sang vị trí khác mà không truyền dữ liệu đó qua một thanh ghi trước.

**Thanh ghi đa dụng (General Purpose Registers)**
Loại thanh ghi đầu tiên được gọi là Thanh ghi đa dụng (GPR). GPR được gọi là “đa dụng” vì chúng có thể chứa dữ liệu, trong trường hợp này lên tới giá trị 64-bit, hoặc địa chỉ bộ nhớ (một con trỏ). Một giá trị trong GPR có thể được xử lý bằng các phép toán như cộng, nhân, dịch bit, v.v.

Trong hầu hết các sách về assembly, có những chương dài dành cho sự tinh tế của GPR, bối cảnh lịch sử của chúng, v.v. Điều này là vì GPR rất quan trọng trong lập trình hệ điều hành, đảo ngược (reverse engineering), v.v. Trong mã assembly viết trong FFmpeg, GPR giống như giàn giáo (scaffolding), và phần lớn thời gian sự phức tạp của chúng không cần thiết và được trừu tượng hóa đi.

**Thanh ghi vectơ**
Thanh ghi vectơ (SIMD), đúng như tên gọi, chứa nhiều phần tử dữ liệu. Có nhiều loại thanh ghi vectơ:

* thanh ghi mm – thanh ghi MMX, kích thước 64-bit, mang tính lịch sử và ít dùng hiện nay
* thanh ghi xmm – thanh ghi XMM, kích thước 128-bit, phổ biến rộng rãi
* thanh ghi ymm – thanh ghi YMM, kích thước 256-bit, có một số phức tạp khi sử dụng
* thanh ghi zmm – thanh ghi ZMM, kích thước 512-bit, khả dụng hạn chế

Phần lớn phép tính trong nén và giải nén video là dựa trên số nguyên nên chúng ta sẽ tập trung vào đó. Dưới đây là ví dụ 16 bytes trong một thanh ghi xmm:

| a | b | c | d | e | f | g | h | i | j | k | l | m | n | o | p |
| :---- | :---- | :---- | :---- | :---- | :---- | :---- | :---- | :---- | :---- | :---- | :---- | :---- | :---- | :---- | :---- |

Nhưng nó cũng có thể là tám word (số nguyên 16-bit)

| a | b | c | d | e | f | g | h |
| :---- | :---- | :---- | :---- | :---- | :---- | :---- | :---- |

Hoặc bốn doubleword (số nguyên 32-bit)

| a | b | c | d |
| :---- | :---- | :---- | :---- |

Hoặc hai quadword (số nguyên 64-bit):

| a | b |
| :---- | :---- |

Tóm lại:

* **b**ytes – dữ liệu 8-bit
* **w**ords – dữ liệu 16-bit
* **d**oublewords – dữ liệu 32-bit
* **q**uadwords – dữ liệu 64-bit
* **d**ouble **q**uadwords – dữ liệu 128-bit

Các ký tự in đậm sẽ quan trọng ở phần sau.

**Include x86inc.asm**
Bạn sẽ thấy trong nhiều ví dụ chúng ta include tệp x86inc.asm. X86inc.asm là một lớp trừu tượng nhẹ được sử dụng trong FFmpeg, x264 và dav1d để giúp cuộc sống của lập trình viên assembly dễ dàng hơn. Nó hỗ trợ nhiều cách, nhưng trước hết, một điều hữu ích là nó gán nhãn cho các GPR như r0, r1, r2. Điều này có nghĩa là bạn không cần phải nhớ tên thanh ghi. Như đã đề cập trước đó, GPR thường chỉ là giàn giáo nên điều này giúp công việc đơn giản hơn nhiều.

**Một đoạn asm scalar đơn giản**

Hãy xem một đoạn asm scalar đơn giản (và khá nhân tạo) để hiểu điều gì đang diễn ra:

```assembly
mov  r0q, 3
inc  r0q
dec  r0q
imul r0q, 5
```

Ở dòng đầu tiên, *giá trị tức thời* (immediate value) 3 (một giá trị được lưu trực tiếp trong mã assembly thay vì được lấy từ bộ nhớ) được lưu vào thanh ghi r0 dưới dạng quadword. Lưu ý rằng trong cú pháp Intel, toán hạng nguồn (nằm bên phải) được chuyển sang toán hạng đích (nằm bên trái), tương tự như hành vi của memcpy. Bạn cũng có thể đọc nó như “r0q = 3”, vì thứ tự giống nhau. Hậu tố “q” của r0 cho biết thanh ghi được dùng dưới dạng quadword. Lệnh inc tăng giá trị lên nên r0q chứa 4, lệnh dec giảm giá trị trở lại 3. Lệnh imul nhân giá trị với 5. Vì vậy cuối cùng r0q chứa 15.

Lưu ý rằng các lệnh dễ đọc cho con người như mov và inc, được assembler chuyển thành mã máy, được gọi là *mnemonics*. Bạn có thể thấy trên mạng và trong sách các mnemonic được viết bằng chữ hoa như MOV và INC nhưng chúng giống với phiên bản chữ thường. Trong FFmpeg, chúng tôi dùng mnemonic chữ thường và dành chữ hoa cho macro.

**Hiểu một hàm vectơ cơ bản**

Đây là hàm SIMD đầu tiên của chúng ta:

```assembly
%include "x86inc.asm"

SECTION .text

;static void add_values(uint8_t *src, const uint8_t *src2)
INIT_XMM sse2
cglobal add_values, 2, 2, 2, src, src2
    movu  m0, [srcq]
    movu  m1, [src2q]

    paddb m0, m1

    movu  [srcq], m0

    RET
```

Hãy xem từng dòng:

```assembly
%include "x86inc.asm"
```

Đây là một “header” được phát triển trong cộng đồng x264, FFmpeg và dav1d để cung cấp các tiện ích, tên định nghĩa sẵn và macro (như cglobal bên dưới) nhằm đơn giản hóa việc viết assembly.

```assembly
SECTION .text
```

Điều này chỉ ra phần nơi mã thực thi được đặt. Điều này khác với phần .data, nơi bạn có thể đặt dữ liệu hằng số.

```assembly
;static void add_values(uint8_t *src, const uint8_t *src2)
INIT_XMM sse2
```

Dòng đầu là comment (dấu “;” trong asm giống “//” trong C) cho thấy đối số hàm trông như thế nào trong C. Dòng thứ hai cho biết chúng ta khởi tạo hàm để sử dụng thanh ghi XMM với tập lệnh sse2. Điều này là vì paddb là lệnh sse2. Chúng ta sẽ nói chi tiết hơn về sse2 trong bài học tiếp theo.

```assembly
cglobal add_values, 2, 2, 2, src, src2
```

Đây là dòng quan trọng vì nó định nghĩa một hàm C có tên “add_values”.

Hãy xem từng phần:

* Tham số tiếp theo cho biết hàm có hai đối số.
* Tham số sau đó cho biết chúng ta sẽ sử dụng hai GPR trong hàm này, bao gồm cả các đối số. Trong một số trường hợp chúng ta có thể muốn dùng nhiều GPR hơn nên phải thông báo cho x86util.
* Tham số tiếp theo cho x86util biết chúng ta sẽ sử dụng bao nhiêu thanh ghi XMM.
* Hai tham số tiếp theo là nhãn cho các đối số của hàm.

Cũng cần lưu ý rằng mã cũ có thể không có nhãn cho các đối số hàm mà truy cập trực tiếp GPR như r0, r1, v.v.

```assembly
    movu  m0, [srcq]
    movu  m1, [src2q]
```

movu là dạng rút gọn của movdqu (move double quad unaligned). Vấn đề căn chỉnh sẽ được nói trong bài khác nhưng hiện tại có thể xem movu là thao tác chuyển 128-bit
