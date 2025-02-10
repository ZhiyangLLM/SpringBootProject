# SpringBootProject
记录项目开发过程中遇到的问题及新学到的内容  
#### 图片压缩处理  
>项目对接客户提出新的需求 获取到的图像文件基本不会在100K以内，但设备端仅能接收150K大小的图像文件，所以需要服务器端对图片进行裁剪压缩
>常用到的图像压缩处理方法：  
>+ [Java中实现高清图片压缩的两种技术方案（通过改变质量压缩图片）]<https://blog.csdn.net/weixin_32005771/article/details/143544417>  
>+ [Linux命令行下进行图片压缩，常用方法：ImageMagick]<https://worktile.com/kb/ask/418626.html>  
>+ [Java Graphics2D库和Thumbnailator库]<https://juejin.cn/post/7206602224042197052?searchId=20241226150321FA8397977FF44C3576EC>
>  
>为了减少代码量、方便后期维护，选择了Thumbnailator库进行图片压缩
>
    /**
     * 压缩图片
     * @param picture
     * @throws IOException
     */
    public static void pictureCompression(File picture) throws IOException {
        BufferedImage originalImage = ImageIO.read(picture);
        int targetWidth = 480;
        int targetHeight = 640;
        // 计算缩放比例，确保宽高不低于480*680
        double scale = Math.max((double)targetWidth / originalImage.getWidth(), (double)targetHeight / originalImage.getHeight());//压缩比例
        Thumbnails.of(picture)  // 输入图片
                .scale(scale)  //设置比例
                .toFile(picture); // 输出到文件
        if (picture.length() > FACE_FILE_SIZE) {
            //进一步降低图片质量
            reduceQuality(picture, FACE_FILE_SIZE); // 压缩到小于等于150KB
        }
    }

    /**
     * 通过降低图片质量压缩图片
     * @param picture
     * @param targetSize
     * @throws IOException
     */
    private static void reduceQuality(File picture, long targetSize) throws IOException {
        // 初始压缩质量设置为 0.8
        float quality = 0.8f;
        while (true) {
            Thumbnails.of(picture)
                    .scale(1)
                    .outputQuality(quality)  // 设置压缩质量
                    .toFile(picture);
            // 获取压缩后的文件大小
            long currentSize = picture.length();
            // 如果文件大小小于等于目标大小，退出循环
            if (currentSize <= targetSize) {
                break;
            } else {
                // 如果文件过大，降低质量
                quality -= 0.05f; // 每次降低质量
                if (quality < 0.1) {
                    break;  // 避免质量过低
                }
            }
        }
    }
#### 输入条件筛选数据，导出时仍为全部数据  
>查询和导出方法如下
>
    private LambdaQueryWrapper<SysLogLoginEntity> getWrapper(SysLogLoginQuery query, UserDetail userDetail) {

        LambdaQueryWrapper<SysLogLoginEntity> wrapper = Wrappers.lambdaQuery();
        wrapper.like(StrUtil.isNotBlank(query.getUsername()), SysLogLoginEntity::getUsername, query.getUsername());
        wrapper.like(StrUtil.isNotBlank(query.getIp()), SysLogLoginEntity::getIp, query.getIp());
        wrapper.eq(SysLogLoginEntity::getComId, userDetail.getComId());
        RoleTypeEnum roleTypeEnum = RoleTypeEnum.getByCode(userDetail.getRoleType());
        if (RoleTypeEnum.PROJECT_COMMUNITY.equals(roleTypeEnum)
                || RoleTypeEnum.PROJECT_OFFICE.equals(roleTypeEnum)) {
            wrapper.eq(SysLogLoginEntity::getUserId, userDetail.getId());
        }
        wrapper.orderByDesc(SysLogLoginEntity::getId);

        return wrapper;
    }
    @Override
    @SneakyThrows
    public void export(SysLogLoginQuery query) {
        UserDetail detail = SecurityUser.getUser();
        if (Objects.isNull(detail)) {
            return;
        }

        LambdaQueryWrapper<SysLogLoginEntity> wrapper = getWrapper(query, detail);
        List<SysLogLoginEntity> list = list(wrapper);

        SysTimezoneEntity zone = timezoneDao.selectById(detail.getTimezoneId());
        String timeOffSet = Objects.isNull(zone) ? null : zone.getTimeOffset();
        List<SysLogLoginVO> sysLogLoginVOS = CollectionUtil.newArrayList();
        for (SysLogLoginEntity loginLog : list) {
            SysLogLoginVO logLoginVO = SysLogLoginConvert.INSTANCE.convert(loginLog);
            logLoginVO.setCreateTime(DateTimeUtil.translate(loginLog.getCreateTime(), timeOffSet));
            sysLogLoginVOS.add(logLoginVO);
        }
        // 写到浏览器打开
        ExcelUtils.excelExport(SysLogLoginVO.class, "system_login_log_excel" + DateUtils.format(new Date()), null, sysLogLoginVOS);

    }
>deubg发现，在查询时，query能正确获取到传入的数据，在export时，query为空，即数据传输有问题  
>返回controller检查数据传输
>
    @GetMapping("export")
    @Operation(summary = "导出excel")
    @OperateLog(type = OperateTypeEnum.EXPORT)
    @PreAuthorize("hasAuthority('sys:log:login')")
    public void export(SysLogLoginQuery query) {
        sysLogLoginService.export(query);
    }
>缺了@RequestBody注解
>
    @PostMapping("export")
    @Operation(summary = "导出excel")
    @OperateLog(type = OperateTypeEnum.EXPORT)
    @PreAuthorize("hasAuthority('sys:log:login')")
    public void export(@RequestBody  SysLogLoginQuery query) {
        sysLogLoginService.export(query);
    }
>添加@RequestBody后成功根据筛选结果导出数据
>>@RequestBody主要用来接收前端传递给后端的json字符串中的数据(请求体中的数据)

>另外 请求体是 JSON 格式的对象，所以修改前后端请求为post（原本为get)
