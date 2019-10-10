# Домашнее задание к лекции "Android: Работа с камерой, Notifications API"
Для каждой задачи создайте решение на базе Gradle и залейте его в GitHub.
К заданию должен быть прикреплен сервер.

## Задача №1 - реализовать отправку фото
В этой лекции мы полностью реализовали отправку фото на сервер.
Ваша задача реализовать это у себя в приложении.

API
```kotlin
// API.kt
...
  @Multipart
  @POST("api/v1/media")
  suspend fun uploadImage(@Part file: MultipartBody.Part):
  Response<AttachmentModel>
...
```
Модель attachment-а
```kotlin
data class AttachmentModel(val id: String, val mediaType: AttachmentType) {
  val url
    get() = "$BASE_URL/api/v1/static/$id"
}
```


Репозиторий
```kotlin
// Repository.kt
...
  suspend fun upload(bitmap: Bitmap): Response<AttachmentModel> {
    // Создаем поток байтов 
    val bos = ByteArrayOutputStream()
    // Помещаем Bitmap в качестве JPEG в этот поток
    bitmap.compress(Bitmap.CompressFormat.JPEG, 100, bos)
    val reqFIle =
    // Создаем тип медиа и передаем массив байтов с потока
    RequestBody.create(MediaType.parse("image/jpeg"), bos.toByteArray())
    val body = 
    // Создаем multipart объект, где указываем поле, в котором
    // содержатся посылаемые данные, имя файла и медиафайл
    MultipartBody.Part.createFormData("file", "image.jpg", reqFIle)
    return API.uploadImage(body)
  }
```
Создание поста
```kotlin
// CreatePostAcivity.kt
...
  val REQUEST_IMAGE_CAPTURE = 1
  private var dialog: ProgressDialog? = null
  private var attachmentModel: AttachmentModel? = null
...
    attachPhotoImg.setOnClickListener {
      dispatchTakePictureIntent()
    }
    createPostBtn.setOnClickListener {
 ...
          val result =
          Repository.createPost(contentEdt.text.toString(), attachmentModel)
          ...

  override fun onActivityResult(requestCode: Int,
  resultCode: Int, data: Intent?) {
    if (requestCode == REQUEST_IMAGE_CAPTURE && resultCode == RESULT_OK) {
      val imageBitmap = data?.extras?.get("data") as Bitmap?
      imageBitmap?.let {
        launch {
          dialog = createProgressDialog()
          val imageUploadResult = Repository.upload(it)
          dialog?.dismiss()
          if (imageUploadResult.isSuccessful) {
            imageUploaded()
            attachmentModel = imageUploadResult.body()
          } else {
            toast("Can't upload image")
...

```
Создание поста(продолжение)
```kotlin
// CreatePostActivity.kt
 private fun imageUploaded() {
    transparetAllIcons()
    // Показываем красную галочку над фото
    attachPhotoDoneImg.visibility = View.VISIBLE
  }

  // Устанавливаем все иконки в полупрозрачный серый цвет
  private fun transparetAllIcons() {
    attachPhotoImg.setImageResource(R.drawable.ic_add_a_photo_inactive)
    attachAudioImg.setImageResource(R.drawable.ic_add_an_audio_inactive)
    attachVideoImg.setImageResource(R.drawable.ic_add_a_video_inactive)
  }

  // Отправка intent-а для приложения камеры
  private fun dispatchTakePictureIntent() {
    Intent(MediaStore.ACTION_IMAGE_CAPTURE).also { takePictureIntent ->
      takePictureIntent.resolveActivity(packageManager)?.also {
        startActivityForResult(takePictureIntent, REQUEST_IMAGE_CAPTURE)
      }
      ...
```
Результаты
| До выбора медиа | После выбора фото|
| -------- | -------- |
|  ![image alt](https://i.imgur.com/FyM2LtP.png =200x350)  | ![image alt](https://i.imgur.com/2xEvVkB.pnghttps://i.imgur.com/gguExx3.png =200x350) |
Один из способов вставить изображение
```xml
//item_post.xml
...
    <FrameLayout
        android:id="@+id/mediaContainer"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:padding="16dp"
        app:layout_constraintBottom_toTopOf="@id/likeBtn"
        app:layout_constraintEnd_toEndOf="parent"
        app:layout_constraintStart_toStartOf="parent"
        app:layout_constraintTop_toBottomOf="@id/contentTv">

        <ImageView
            android:id="@+id/photoImg
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:adjustViewBounds="true"
            tools:src="@drawable/example"
            app:layout_constraintHeight_max="400dp" />
    </FrameLayout>
```
Загрузка в адаптере
```kotlin
// PostAdapter.kt
      when (post.attachment?.mediaType) {
        AttachmentType.IMAGE -> loadImage(photoImg, post.attachment.url) }
        ...
        
    private fun loadImage(photoImg: ImageView, imageUrl: String) {
    Glide.with(photoImg.context)
      .load(imageUrl)
      .into(photoImg)
  }

```
Результат
![image alt](https://i.imgur.com/Ktug15W.png)


___

## Задача №2 - добавить нотификацию в приложение.
Давайте после выхода из приложения добавим нотификацию о том, что мы будем рады видеть пользователя в приложении еще раз. Делать это будем только первый раз.
Для этого на понадобиться флаг, который будет сообщать нам о том, перый раз мы в приложении или нет. Записывать и считывать флаг мы конечно же будем с помощью `SharedPreferences`

Записывать будем в тот же файл, что использовался при записи токена.
Добавим ключ для нашей константы:
```kotlin=
//Constants.kt
const val FIRST_TIME_SHARED_KEY = "first_time_shared_key"
```
Реализуем методы записи и считывания файле Utils (потому что эта информация в дальнейшем может понадобиться во многих частях приложения).

```kotlin=
// Utils.kt
fun isFirstTime(context: Context) =
  context.getSharedPreferences(API_SHARED_FILE, Context.MODE_PRIVATE).getBoolean(
    FIRST_TIME_SHARED_KEY, true
  )

fun setNotFirstTime(context: Context) =
  context.getSharedPreferences(API_SHARED_FILE, Context.MODE_PRIVATE)
    .edit()
    .putBoolean(API_SHARED_FILE, false)
```

Теперь после выхода из главного экрана (вызовется колбек `onDestroy()`)
```kotlin=
  override fun onDestroy() {
    super.onDestroy()
    if (isFirstTime(this)) {
      NotifictionHelper.comeBackNotification(this)
      setNotFirstTime(this)
    }
  }
```
Вам остается реализоват метод `NotifictionHelper.comeBackNotification(this)`
![](https://i.imgur.com/jfJPKL6.jpg)


# Задача № 3 (необязательная) - Jobscheduler и нотификации