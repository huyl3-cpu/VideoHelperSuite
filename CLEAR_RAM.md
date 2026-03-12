# clear_ram — VideoHelperSuite

## Chức năng

`clear_ram` là một tùy chọn boolean trong node **VideoCombine**. Khi bật, nó sẽ **xóa tensor image frames khỏi RAM** ngay sau khi video đã được ghi xong ra disk.

## Mục đích

Khi tạo video dài với các node như **GIMM-VFI**, **SeedVR2**, **WanVideo**, hàng nghìn frame ảnh được giữ trong RAM trong suốt quá trình combine. Sau khi video đã lưu xong, các tensor này không còn cần thiết nữa và chiếm dung lượng RAM lớn.

`clear_ram = True` tự động giải phóng bộ nhớ đó.

## Phạm vi xóa — chỉ image tensors

| Loại bộ nhớ | Bị xóa? |
|-------------|---------|
| Image frames tensor (input `images`) | ✅ Có — giải phóng sau khi lưu xong |
| Model weights trong VRAM | ❌ Không |
| CUDA cache (`torch.cuda.empty_cache`) | ❌ Không |
| Job history / queue metadata ComfyUI | ❌ Không |
| Video file đã lưu ra disk | ❌ Không |
| ComfyUI execution cache của node khác | ❌ Không |

## Cơ chế hoạt động

```python
# Sau khi combine_video() lưu file xong:
if clear_ram:
    if images_tensor_ref is not None:
        del images_tensor_ref   # xóa reference đến tensor frames
        gc.collect()            # Python GC giải phóng bộ nhớ ngay
```

- `del` xóa reference Python → tensor frames không còn ai giữ → eligible for GC
- `gc.collect()` kích hoạt garbage collector ngay lập tức thay vì chờ tự động
- Chỉ ảnh hưởng đến tensor `images` được truyền vào node — không đụng gì khác

## Khi nào nên bật

- Workflow có nhiều frame (>100 frames)
- Dùng model upscale/interpolate tạo ra lượng lớn frame trong RAM
- Chạy nhiều job liên tiếp và cần giải phóng RAM giữa các job

## Lưu ý

- Job history và queue ComfyUI **không bị ảnh hưởng** (history lưu metadata, không lưu tensor)
- Nếu node sau trong workflow cần reuse tensor `images`, **tắt** `clear_ram`
- `clear_ram` chỉ hoạt động khi input là `torch.Tensor` (không hoạt động với latent dict)
