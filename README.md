import com.android.build.gradle.api.ApplicationVariant
import org.apache.commons.compress.utils.IOUtils
import org.objectweb.asm.ClassReader
import org.objectweb.asm.ClassVisitor
import org.objectweb.asm.ClassWriter
import org.objectweb.asm.FieldVisitor
import org.objectweb.asm.MethodVisitor
import org.objectweb.asm.Opcodes
import org.objectweb.asm.Type

import java.util.jar.JarEntry
import java.util.jar.JarFile
import java.util.jar.JarOutputStream

//引用插件
apply plugin: 'com.android.application'


//扩展
android {
    compileSdkVersion 28
    buildToolsVersion "29.0.0"
    defaultConfig {
        applicationId "com.enjoy.hotfix"
        minSdkVersion 14
        targetSdkVersion 28
        versionCode 1
        versionName "1.0"
        testInstrumentationRunner "android.support.test.runner.AndroidJUnitRunner"
    }

    buildTypes {
        release {
            minifyEnabled false
            proguardFiles getDefaultProguardFile('proguard-android-optimize.txt'), 'proguard-rules.pro'
        }
    }
}

dependencies {
    implementation fileTree(dir: 'libs', include: ['*.jar'])
    implementation 'com.android.support:appcompat-v7:28.0.0'
    implementation 'com.android.support:appcompat-v7:28.0.0'
    implementation 'com.android.support.constraint:constraint-layout:1.1.3'
    testImplementation 'junit:junit:4.12'
    androidTestImplementation 'com.android.support.test:runner:1.0.2'
    androidTestImplementation 'com.android.support.test.espresso:espresso-core:3.0.2'

    implementation project(':lib')
}


//gradle执行会解析build.gradle文件，afterEvaluate表示在解析完成之后再执行我们的代码
afterEvaluate({
    android.getApplicationVariants().all {
        variant ->
            //获得: debug/release
            String variantName = variant.name
            //首字母大写 Debug/Release
            String capitalizeName = variantName.capitalize()

            //这就是打包时，把jar和class打包成dex的任务
            Task dexTask =
                    project.getTasks().findByName("transformClassesWithDexBuilderFor" + capitalizeName);

            //在他打包之前执行插桩
            dexTask.doFirst {
                //任务的输入，dex打包任务要输入什么？ 自然是所有的class与jar包了！
                FileCollection files = dexTask.getInputs().getFiles()

                for (File file : files) {
                    //.jar ->解压-》插桩->压缩回去替换掉插桩前的class
                    // .class -> 插桩
                    String filePath = file.getAbsolutePath();
                    //依赖的库会以jar包形式传过来，对依赖库也执行插桩
                    if (filePath.endsWith(".jar")) {
                        processJar(file);

                    } else if (filePath.endsWith(".class")) {
                        //主要是我们自己写的app模块中的代码
                        processClass(variant.getDirName(), file);
                    }
                }
            }
    }
})


static boolean isAndroidClass(String filePath) {
    return filePath.startsWith("android") ||
            filePath.startsWith("androidx");
}

static byte[] referHackWhenInit(InputStream inputStream) throws IOException {
    // class的解析器
    ClassReader cr = new ClassReader(inputStream)
    // class的输出器
    ClassWriter cw = new ClassWriter(cr, 0)
    // class访问者，相当于回调，解析器解析的结果，回调给访问者
    ClassVisitor cv = new ClassVisitor(Opcodes.ASM5, cw) {

        //要在构造方法里插桩 init
        @Override
        public MethodVisitor visitMethod(int access, final String name, String desc,
                                         String signature, String[] exceptions) {

            MethodVisitor mv = super.visitMethod(access, name, desc, signature, exceptions);
            mv = new MethodVisitor(Opcodes.ASM5, mv) {
                @Override
                void visitInsn(int opcode) {
                    //在构造方法中插入AntilazyLoad引用
                    if ("<init>".equals(name) && opcode == Opcodes.RETURN) {
                        //引用类型
                        //基本数据类型 : I J Z
                        super.visitLdcInsn(Type.getType("Lcom/enjoy/patch/hack/AntilazyLoad;"));
                    }
                    super.visitInsn(opcode);
                }
            };
            return mv;
        }

    };
    //启动分析
    cr.accept(cv, 0);
    return cw.toByteArray();
}

/**
 * linux/mac: /xxxxx/app/build/intermediates/classes/debug/com/enjoy/qzonefix/MainActivity.class
 * windows: \xxxxx\app\build\intermediates\classes\debug\com\enjoy\qzonefix\MainActivity.class
 * @param file
 * @param hexs
 */
static void processClass(String dirName, File file) {

    String filePath = file.getAbsolutePath();
    //注意这里的filePath包含了目录+包名+类名，所以去掉目录
    String className = filePath.split(dirName)[1].substring(1);
    //application或者android support我们不管
    if (className.startsWith("com\\enjoy\\hotfix\\MyApplication") || isAndroidClass(className)) {
        return
    }

    try {
        // byte[]->class 修改byte[]
        FileInputStream is = new FileInputStream(filePath);
        //执行插桩  byteCode:插桩之后的class数据，把他替换掉插桩前的class文件
        byte[] byteCode = referHackWhenInit(is);
        is.close();

        FileOutputStream os = new FileOutputStream(filePath)
        os.write(byteCode)
        os.close()
    } catch (Exception e) {
        e.printStackTrace();
    }
}


static void processJar(File file) {
    try {
        //  无论是windows还是linux jar包都是 /
        File bakJar = new File(file.getParent(), file.getName() + ".bak");
        JarOutputStream jarOutputStream = new JarOutputStream(new FileOutputStream(bakJar));

        JarFile jarFile = new JarFile(file);
        Enumeration<JarEntry> entries = jarFile.entries();
        while (entries.hasMoreElements()) {
            JarEntry jarEntry = entries.nextElement();

            // 读jar包中的一个文件 ：class
            jarOutputStream.putNextEntry(new JarEntry(jarEntry.getName()));
            InputStream is = jarFile.getInputStream(jarEntry);

            String className = jarEntry.getName();
            if (className.endsWith(".class") && !className.startsWith
                    ("com/enjoy/hotfix/MyApplication")
                    && !isAndroidClass(className) && !className.startsWith("com/enjoy" +
                    "/patch")) {
                byte[] byteCode = referHackWhenInit(is);
                jarOutputStream.write(byteCode);
            } else {
                //输出到临时文件
                jarOutputStream.write(IOUtils.toByteArray(is));
            }
            jarOutputStream.closeEntry();
        }
        jarOutputStream.close();
        jarFile.close();
        file.delete();
        bakJar.renameTo(file);
    } catch (IOException e) {
        e.printStackTrace();
    }
}
