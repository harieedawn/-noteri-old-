# NOTERI

Ứng dụng ghi chú và học tập chạy trên trình duyệt (PWA), cài bằng Chrome. Một trang là một tờ giấy vô hạn để vẽ tay, gõ chữ, chèn ảnh, ghi chú dán, vẽ sơ đồ tư duy; PDF được chú thích không phá hủy. Mọi dữ liệu nằm trên máy người dùng.

Tài liệu này dành cho người và AI tiếp tục phát triển. Đọc hết mục "Quyết định của chủ dự án" và "Lỗi đã từng mắc" trước khi sửa code.

## Mục tiêu sản phẩm
- Cảm giác như GoodNotes / Notability: viết mượt, zoom mượt, không giật, không mất dữ liệu.
- Giao diện phẳng, bo góc mọi nơi, kính mờ (blur) được phép, hiệu ứng nhẹ. Tiếng Việt.
- Phông chữ toàn ứng dụng là Google Sans qua biến CSS `--ff` (kể cả chữ trong trang và chữ NOTERI ở màn hình mở đầu). Không viết `system-ui` hay tên phông cứng ở nơi khác; dùng `var(--ff)`. Đo chữ bằng canvas (`lnBase`) phải dùng cùng phông, và dựng lại khi phông tải xong (`document.fonts.ready`).
- Chạy offline sau lần tải đầu. Không tài khoản, không quảng cáo, không gửi dữ liệu đi.
- Dùng được một tay trên điện thoại (dọc và ngang), tablet có bút.

## Cấu trúc
- `index.html`: toàn bộ ứng dụng (CSS, HTML, JS trong một file, không build). `sw.js`: service worker (stale-while-revalidate, đổi tên cache `noteri-vN` mỗi bản). `manifest.webmanifest`, `icon-192.png`, `icon-512.png`, `fonts/GoogleSans-var.woff` (Google Sans biến thiên, trục chữ đậm 400-700, đã cắt còn Latin + tiếng Việt, kèm `fonts/OFL.txt`). Thêm tệp tĩnh mới thì nhớ thêm vào danh sách `A` trong `sw.js`.
- Thư viện ngoài duy nhất: pdf.js 3.11.174 từ cdnjs (đọc PDF). Không thêm framework.
- Hàm tiện ích: `h(tag,attrs,...children)` tạo DOM; `ic(name,size)` trả SVG; `IC` là bảng icon (nét 1.7, bo tròn); `$` = querySelector.

## Dữ liệu (IndexedDB `noteri`, store `k`)
- `meta`: `{folders:[{id,name,color,ic,parent}],docs:[{id,name,type:'page'|'pdf',folder,updated,paper,ratio,txt,th,np,deleted}]}`. `txt` là văn bản chuẩn hóa để tìm kiếm không dấu; `th` là ảnh xem trước.
- `d:<id>`: JSON chuỗi `{ratio,pattern,sp,paper,strokes,objs,edges}`. `p:<id>`: ArrayBuffer PDF gốc (không bao giờ sửa).
- Cài đặt người dùng: `localStorage` `noteri.t` (đối tượng `T`), `noteri.col`, `noteri.den`.
- Nét vẽ: `{t:'p'|'h',c,w,p:[x,y,...],q:[hệ số bề rộng theo điểm]|null}`. Tọa độ theo đơn vị trang (cạnh dài 2400). Nét cũ không có `q` vẫn phải đọc được.
- `objs` (đối tượng, thứ tự = lớp): `img`, `text`, `note`, `node`, `group`. Trường chung `{id,k,x,y,w,h,r,txt,fs,c,c2,op,lock}`. Ảnh lưu `src` (dataURL) nhưng `src` bị loại khỏi snapshot lịch sử (xem `rep`). `text` có thêm `sn:1` khi bật hít chữ vào dòng kẻ. `group` dùng `txt` làm tên nhóm, không lưu danh sách thành viên (xem mục Sơ đồ tư duy).
- `edges` (mũi tên sơ đồ): `{id,a,b,st:'curve'|'line'|'elbow',ar:0|1,w?,c?}`; `w` là độ dày theo đơn vị trang (thiếu thì dùng `EW0`=6); `a`,`b` là id của `node`. Điểm nối được tính mỗi khung hình từ hình học node, không lưu.
- Tương thích ngược là bắt buộc: không đổi tên trường, không làm hỏng trang cũ. Muốn đổi định dạng thì đọc được cả hai.

## Kiến trúc trình soạn thảo
- Hệ tọa độ: một "đơn vị trang" = 1px của thế giới. Biến đổi `S.tr={s,x,y}` (tỉ lệ + dịch). `W2(x,y)` màn hình -> thế giới. DOM của đối tượng nằm trong `#world` (một CSS transform duy nhất); nét vẽ nằm trên canvas `c1`; giấy, PDF và mũi tên sơ đồ nằm trên canvas `c0` (dưới đối tượng DOM).
- Thứ tự lớp: giấy/PDF/mũi tên (`c0`) -> đối tượng DOM -> nét vẽ + khung chọn (`c1`).
- Một bộ cử chỉ duy nhất trên `#stage` (pointer events): hai ngón luôn pinch/pan với mọi công cụ; một ngón theo công cụ; bút (stylus) vẽ còn ngón tay cuộn; chống chạm nhầm lòng bàn tay.
- Lịch sử hoàn tác: `touch()` chụp snapshot trước thay đổi, `commit()` đẩy vào ngăn xếp. Snapshot = `{s: mảng tham chiếu stroke, o: JSON objs (không src), e: JSON edges}`. Vì vậy stroke đã commit là bất biến (xóa từng phần tạo stroke mới, không sửa stroke cũ).
- Hiệu năng: khi hơn 150 nét, các nét đã commit được cache vào bitmap lớn hơn màn hình 25% mỗi phía (`icBuild`/`icAdd`); chỉ dựng lại khi `S.inkV` đổi. PDF render lười, tối đa 2 trang cùng lúc, ngân sách 36 triệu điểm ảnh.
- Lưu: `dirty()` hẹn 800ms rồi `flush()`; cũng lưu khi ẩn trang. Ảnh xem trước tạo ở `flush(true)`.
- Điều hướng: `history.pushState` cho mỗi cấp thư mục và khi mở trình soạn; nút quay lại của điện thoại lùi từng cấp, ở trang chủ mới thoát. Không thay đổi `cur` mà không đẩy history.

## Sơ đồ tư duy
- Công cụ `mm` nằm trên thanh công cụ. Nó là chế độ con của công cụ Chọn (`S.mm=true`, `S.tool==='select'`), nên chọn, kéo, đổi cỡ, nhân bản, khóa đều dùng chung mã với đối tượng khác. Nhấn lần nữa (hoặc giữ) để mở bảng chế độ.
- Chế độ "Thêm nút": chạm chỗ trống tạo nút và vào gõ chữ. Chế độ "Nối nút": kéo từ nút này sang nút khác để tạo mũi tên. Khi nút được chọn có 4 dấu cộng ở các cạnh: chạm = thêm nút con theo hướng đó; kéo sang nút khác = nối; kéo ra chỗ trống = tạo nút mới và nối.
- Nút có `h` tự co theo nội dung; chỉ đổi độ rộng (tay nắm góc dưới phải), không xoay. Xóa nút xóa luôn mũi tên liên quan. Mũi tên có thể chọn bằng cách chạm gần đường; thanh thao tác đổi kiểu, đầu mũi tên, xóa.
- Mũi tên (`mmDraw`): nét mảnh, độ dày `w` (2 đến 24, đơn vị trang). Đầu mũi tên tỉ lệ theo `w` (`max(20, 3.8w)`), đường dừng ở gốc đầu mũi tên; đường gấp khúc bo góc. Chọn mũi tên thì thanh thao tác có thanh trượt độ dày; bảng Sơ đồ tư duy có thanh trượt đặt độ dày mặc định cho mũi tên mới (`T.mm.w`).
- Nhóm (kiểu Obsidian), obj `k:'group'`: khung mờ bo góc, tên nằm phía trên (chip). Luôn nằm dưới cùng của `objs` (`gfix` sắp nhóm lớn xuống dưới nhóm nhỏ). Thành viên được tính theo hình học: đối tượng có tâm nằm trong khung (`grpKids`), không lưu danh sách. Chỉ viền (`grpHit`) và nhãn tên bắt được chạm; vùng trong để trống để vẫn tạo nút, kéo, cuộn bên trong nhóm. Kéo viền/tên = di chuyển cả nhóm; nhấn đúp vào tên = đổi tên; tay nắm góc dưới phải = đổi kích thước (không xoay, không nhân bản, không đổi lớp). Xóa nhóm chỉ xóa khung, giữ nút.
- Tạo nhóm: kéo một nút (chế độ Chọn hoặc Thêm nút) sao cho tâm của nó nằm trong nút khác; `ghUpd` bật `S.gh` và `ghDraw` vẽ gợi ý (khung dự kiến nét đứt + vòng quanh nút đích nhấp nháy nhẹ, không có chữ). Bỏ ra ngoài thì gợi ý tắt. Thả tay trong lúc có gợi ý: `objDrop` tạo nhóm (`mkGroup`, nút được xếp cạnh nút đích bởi `grpLay` để không chồng) rồi vào nhập tên. Tên bắt buộc: để trống thì hủy cả thao tác kéo (về trạng thái trước khi kéo). Đích đang nằm sẵn trong nhóm thì không gợi ý; thả nút vào trong nhóm chỉ mở rộng khung nếu nút lòi ra ngoài (`grpGrow`). Cả thao tác là một bước hoàn tác.
- Sửa chữ nhất quán: hộp chữ, ghi chú, nút sơ đồ và tên nhóm đều phải nhấn đúp mới sửa; chạm một lần chỉ chọn (kể cả khi đang ở công cụ Chữ hoặc Ghi chú; ảnh thì công cụ Chữ vẫn tạo chữ đè lên ảnh).
- Hít chữ vào dòng kẻ ngang: nút `#tsn` (góc phải trên) chỉ hiện khi chọn công cụ Chữ; trạng thái lưu ở `T.sn`; mờ đi nếu trang không có kẻ ngang/ô vuông (PDF và giấy trắng thì không có tác dụng). Chữ tạo ra khi đang bật có `sn:1`: đường cơ sở dòng đầu đặt lên đường kẻ gần nhất (`lnSnap`), chiều cao dòng là bội của khoảng kẻ (`lnLH`) nên các dòng sau cũng nằm trên kẻ. Khi kéo chữ có `sn` chỉ hít nhẹ: ngưỡng tối đa 22% khoảng kẻ và không quá 11px màn hình. Chỉ hít theo trục dọc của trang (kẻ ngang); không hít vào đường kẻ dọc của ô vuông; việc căn giữa trang và căn theo đối tượng khác vẫn là `snap()` cũ.
- "Sắp xếp tự động": bố cục cây trái sang phải (`mmLayout`), một bước hoàn tác. Chưa cập nhật khung nhóm theo vị trí mới của nút.
- Có thể vẽ tay, thêm mũi tên tự do bằng bút thường, hoặc trộn với sơ đồ trên cùng một trang. Không tách thành loại tài liệu riêng.
- Chưa có: thu gọn nhánh, nhãn trên mũi tên, màu nhóm, nhóm lồng nhau được kiểm thử kỹ, kéo thả đổi cha, kiểu nút khác, xuất ảnh sơ đồ, vẽ nút/mũi tên/nhóm trong ảnh xem trước.

## Quyết định của chủ dự án (không được đảo ngược nếu chưa hỏi)
- Không dùng gradient làm nền hay nút (đường kẻ trang giấy dùng điểm dừng cứng là chấp nhận được). Không dùng hiệu ứng phát sáng (glow). Blur/kính mờ thì được.
- Không emoji. Không dùng từ "balo" trong chữ hiển thị. Không có chế độ giao diện "Theo máy"; chỉ Sáng, Tối và mục "Thêm" (màu khác, ô xem trước không có tên).
- Không hiện thông báo (toast) cho thao tác đơn giản như bật khóa trang.
- Khóa trang (nút ổ khóa khi xem): bật thì cuộn thẳng tắp theo chiều dọc, chỉ nhích ngang rất nhẹ rồi thu về giữa; tắt thì tự do hoàn toàn. Mặc định tắt.
- Nhấn đúp hoặc nút "vừa khung" chỉ đưa zoom về mặc định, giữ nguyên vị trí đang xem (không nhảy về trang đầu).
- Giới hạn kéo trang tính theo kích thước trang (không theo mức zoom) để khoảng trống kéo được bằng nhau ở mọi mức zoom. Zoom nhỏ nhất của PDF là 0.3 lần mức vừa chiều rộng.
- Chạm hai ngón = hoàn tác, ba ngón = làm lại.
- Màn hình chính: lưới 1 đến 5 cột chụm ngón để đổi, mỗi thư mục nhớ mức riêng; mục có thể thu gọn. Không có mục "Gần đây". Thư mục lồng nhau, có biểu tượng riêng.
- Giọng văn trong "Giới thiệu" và "Có gì mới": chuyên nghiệp, liệt kê gạch đầu dòng, chỉ nêu đúng điều đã thay đổi. Không khoa trương.

- Thanh công cụ: chọn Con trỏ thì thu gọn còn một nút ở giữa (chạm lại để mở rộng). Cả 8 nút luôn nằm trong DOM; thu gọn chỉ là thêm lớp `.cp` và CSS co độ rộng các nút còn lại về 0 (có hiệu ứng, nút Con trỏ nảy nhẹ; mở rộng thì các nút hiện lần lượt). Không dựng lại nút mỗi lần gọi `tools()`, nếu không sẽ mất hiệu ứng. Lần đầu mở trình soạn dùng lớp `.nt` để không chạy hiệu ứng. Khi bảng của công cụ (bút, tẩy, sơ đồ) đang mở, chạm vào trang chỉ đóng bảng, không vẽ.
- Tạo nút sơ đồ hoặc hộp chữ rồi thoát/đóng bàn phím mà chưa nhập gì thì hủy việc tạo (không để lại nút rỗng). Nhóm mới tạo cũng vậy: tên bắt buộc.
- Mọi loại chữ trong trang (hộp chữ, ghi chú, nút, tên nhóm) chỉ sửa bằng nhấn đúp. Hít chữ vào kẻ phải nhẹ, chỉ theo trục ngang của dòng kẻ, không hít mạnh.

## Chuyển động
- Dùng thuộc tính riêng `scale`, `translate`, `rotate`, `opacity` trong keyframes để không đè lên `transform` đặt inline (đối tượng trong trang dùng `transform` để xoay/lật).
- Hoạt ảnh trên phần tử tồn tại lâu dùng `animation-fill-mode: backwards` để tránh giữ giá trị cuối.
- Phần tử mới trong trang chỉ bật lên khi `S.ready` (sau khi mở trang xong); mũi tên sơ đồ vẽ dần theo `ebp()`; vẽ lại theo `S.eba`.
- `prefers-reduced-motion` tắt toàn bộ bằng một luật toàn cục. Không thêm hoạt ảnh lặp lại mỗi lần dựng lại danh sách (gây nháy khi gõ tìm kiếm).

## Lỗi đã từng mắc (đừng lặp lại)
- Dùng đơn vị dp cho phần tử trong trang trong khi phép toán dùng px làm trang lệch góc. Mọi thứ trong trang dùng đơn vị trang.
- Gắn `pointerInput`/handler vào `transform` làm khởi động lại mỗi khung hình. Handler đọc `S.tr` trực tiếp.
- Giới hạn kéo theo pixel màn hình làm zoom lớn gần như không kéo được. Dùng giới hạn theo thế giới (`clampT`).
- Hàm "vừa khung" cho PDF đưa về trang đầu. Phải giữ tọa độ thế giới ở giữa màn hình.
- Lưu `src` ảnh trong snapshot hoàn tác làm bộ nhớ phình và lag. Luôn loại `src` khỏi snapshot.
- Chỉnh stroke đã commit tại chỗ làm hỏng hoàn tác. Tạo stroke mới.
- Render lại PDF khi đang pinch gây giật. Hoãn đến khi thả tay.
- Thêm gradient/glow "cho đẹp". Chủ dự án đã bác bỏ nhiều lần.

## Quy trình phát hành (bắt buộc)
1. Tăng `VER` trong `index.html` (vá lỗi: z; tính năng: y; thay đổi lớn: x).
2. Thêm mục `vc('x.y.z',[...])` ở đầu danh sách "Có gì mới" trong `settings()`: mỗi dòng một thay đổi, thể hiện đúng điều đã làm.
3. Đổi tên cache trong `sw.js` (`noteri-vN`) để máy cũ nhận bản mới.
4. Chạy kiểm tra cú pháp (`node --check` trên phần script) và thử thủ công các luồng bên dưới.
5. Cập nhật README nếu đổi kiến trúc, định dạng dữ liệu hoặc quyết định sản phẩm.

## Kiểm tra thủ công tối thiểu
- Mở trang mới ở 5 tỉ lệ: nằm giữa màn hình. Pinch khi đang chọn bút. Vẽ, hoàn tác, làm lại.
- Chèn ảnh lớn rồi kéo, đổi cỡ, xoay. Chạm công cụ Chữ tạo hộp chữ đúng chỗ chạm.
- Nhập PDF nhiều trang: cuộn, zoom, vẽ ngoài lề, bật/tắt khóa trang, nhấn đúp giữ nguyên vị trí.
- Sơ đồ tư duy: tạo nút, thêm con bằng dấu cộng, kéo nối hai nút, đổi kiểu mũi tên, sắp xếp tự động, hoàn tác từng bước, đóng mở lại vẫn còn.
- Tạo thư mục trong thư mục, nút quay lại điện thoại lùi từng cấp, xoay ngang.

## Hướng phát triển
Thu gọn nhánh sơ đồ, nhãn mũi tên, màu nhóm, xuất PNG/PDF, sao lưu/khôi phục, định dạng chữ Markdown, chọn vùng bằng lasso, bảng chọn đóng bằng nút quay lại điện thoại.
