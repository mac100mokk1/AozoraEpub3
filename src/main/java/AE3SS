import com.github.hmdev.converter.AozoraEpub3Converter;
import com.github.hmdev.image.ImageInfoReader;
import com.github.hmdev.info.BookInfo;
import com.github.hmdev.info.BookInfoHistory;
import com.github.hmdev.info.ProfileInfo;
import com.github.hmdev.info.SectionInfo;
import com.github.hmdev.util.LogAppender;
import com.github.hmdev.web.WebAozoraConverter;
import com.github.hmdev.writer.Epub3ImageWriter;
import com.github.hmdev.writer.Epub3Writer;
import com.github.junrar.Archive;
import com.github.junrar.exception.RarException;
import com.github.junrar.rarfile.FileHeader;
import org.apache.commons.cli.*;
import org.apache.commons.compress.archivers.ArchiveEntry;
import org.apache.commons.compress.archivers.zip.ZipArchiveInputStream;
import org.apache.commons.compress.utils.IOUtils;

import javax.swing.*;
import java.io.*;
import java.net.URL;
import java.text.NumberFormat;
import java.util.LinkedHashMap;
import java.util.Map;
import java.util.Properties;
import java.util.Vector;


/** コマンドライン実行用mainとePub3変換関数 */
public class AE3SS {
	/** 青空→ePub3変換クラス */
	AozoraEpub3Converter aozoraConverter;

	/** Web小説青空変換クラス */
	WebAozoraConverter webConverter;

	/** ePub3出力クラス */
	Epub3Writer epub3Writer;

	/** ePub3画像出力クラス */
	Epub3ImageWriter epub3ImageWriter;

	/** 変換をキャンセルした場合true */
	boolean convertCanceled = false;
	/** 変換実行中 */
	boolean running = false;

	Process kindleProcess;

	/** 設定ファイル */
	Properties props;
	/** 設定ファイル名 */
	String propFileName = "AozoraEpub3.ini";

	/** jarファイルのあるパス文字列 "/"含む */
	String jarPath = null;

	/** 前回の出力パス */
	File currentPath = null;
	/** キャッシュ保存パス */
	File cachePath = null;
	/** Web小説取得情報格納パス */
	File webConfigPath = null;

	/** 選択されているプロファイル */
	ProfileInfo selectedProfile;
	/** プロファイル格納パス */
	File profilePath;
	String presetPath;

	Config config;

	int coverW;
	int coverH;

	public static final String VERSION = "1.1.0b46";

	/** コマンドライン実行用 */
	public static void main(String args[])
	{
		String jarPath = System.getProperty("java.class.path");
		int idx = jarPath.indexOf(";");
		if (idx > 0) jarPath = jarPath.substring(0, idx);
		if (!jarPath.endsWith(".jar")) jarPath = "";
		else jarPath = jarPath.substring(0, jarPath.lastIndexOf(File.separator)+1);
		//this.cachePath = new File(jarPath+".cache");
		//this.webConfigPath = new File(jarPath+"web");

		/** 設定ファイル名 */
		String propFileName = "AozoraEpub3.ini";
		/** 出力先パス */
		File dstPath = null;

		String helpMsg = "AozoraEpub3 [-options] input_files(txt,zip,cbz)\nversion : "+VERSION;

		try {
			//コマンドライン オプション設定
			Options options = new Options();
			options.addOption("h", "help", false, "show usage");
			options.addOption("t", true, "本文内の表題種別\n[0:表題 → 著者名]\n[1:著者名 → 表題]\n[2:表題 → 著者名(副題優先)]\n[3:表題のみ(1行)]\n[4:表題+著者のみ(2行)]\n[5:なし]");
			options.addOption("tf", false, "入力ファイル名を表題に利用");
			options.addOption("c", "cover", true, "表紙画像\n[0:先頭の挿絵]\n[1:ファイル名と同じ画像]\n[2:表紙なし]\n[ファイル名 or URL]");
			options.addOption("ext", true, "出力ファイル拡張子(プロファイル・プリセット優先)\n{.epub, .kepub.epub, .fxl.kepub.epub, .mobi, .mobi+.epub}");
			options.addOption("of", false, "出力ファイル名を入力ファイル名に合せる");
			options.addOption("d", "dst", true, "出力先パス");
			options.addOption("enc", true, "入力ファイルエンコード\n[MS932] (default)\n[UTF-8]");
			//options.addOption("id", false, "栞用ID出力 (for Kobo)");
			//options.addOption("tcy", false, "自動縦中横有効");
			//options.addOption("g4", false, "4バイト文字変換");
			//options.addOption("tm", false, "表題を左右中央");
			//options.addOption("cp", false, "表紙画像ページ追加");
			options.addOption("hor", false, "横書き (指定がなければ縦書き)");
			options.addOption("dev", "preset", true, "プリセット名\n[kindle_pw, kobo__full, kobo_glo, kobo_touch, reader_t3, reader]");
			options.addOption("p", "profile", true, "プロファイル名(優先)\n[/path/to/profile.ini]");

			CommandLine commandLine;
			try {
				commandLine = new BasicParser().parse(options, args, true);
			} catch (ParseException e) {
				new HelpFormatter().printHelp(helpMsg, options);
				return;
			}
			//オプションの後ろをファイル名に設定
			String[] fileNames = commandLine.getArgs();
			if (fileNames.length == 0) {
				new HelpFormatter().printHelp(helpMsg, options);
				return;
			}

			//ヘルプ出力
			if (commandLine.hasOption('h') ) {
				new HelpFormatter().printHelp(helpMsg, options);
				return;
			}
			//iniファイル確認
			if (commandLine.hasOption("i")) {
				propFileName = commandLine.getOptionValue("i");
				File file = new File(propFileName);
				if (file == null || !file.isFile()) {
					LogAppender.error("-i : ini file not exist. "+file.getAbsolutePath());
					return;
				}
			}
			//出力パス確認
			if (commandLine.hasOption("d")) {
				dstPath = new File(commandLine.getOptionValue("d"));
				if (dstPath == null || !dstPath.isDirectory()) {
					LogAppender.error("-d : dst path not exist. "+dstPath.getAbsolutePath());
					return;
				}
			}

			new AE3SS(commandLine, jarPath);

		} catch (Exception e) {
			e.printStackTrace();
		}
	}

	public AE3SS(CommandLine commandLine, String jarPath){
		this.jarPath = jarPath;
		this.cachePath = new File(this.jarPath+".cache");
		this.webConfigPath = new File(this.jarPath+"web");
		this.profilePath = new File(this.jarPath+"profiles");
		this.profilePath.mkdir();
		this.presetPath = jarPath+"presets/";

		//設定ファイル読み込み
		props = new Properties();
		try {
			FileInputStream fos = new FileInputStream(this.jarPath+this.propFileName);
			props.load(fos);
			fos.close();
		} catch (Exception e) { }
		String path = props.getProperty("LastDir");
		if (path != null && path.length() >0) this.currentPath = new File(path);
		this.config = new Config(this);


		if (commandLine.hasOption("t")) {
			try {
				int idx = Integer.parseInt(commandLine.getOptionValue("t"));
				if (0 <= idx && idx <= 5) {
					this.config.jComboTitle = idx;
				}

			} catch (Exception e) {
				LogAppender.println("error at option: -t");
			}
		}
		if (commandLine.hasOption("tf")) {
			this.config.jCheckUseFileName = true;
		}
		if (commandLine.hasOption("c")) {
			if (commandLine.getOptionValue("c").equals("0") || commandLine.getOptionValue("c").equals("1") || commandLine.getOptionValue("c").equals("2")) {
				this.config.setjComboCover(Integer.parseInt(commandLine.getOptionValue("c")));
			} else {
				this.config.setjComboCover(commandLine.getOptionValue("c"));
			}
		}
		if (commandLine.hasOption("ext")) {
			this.config.jComboExt = commandLine.getOptionValue("ext");
		}
		if (commandLine.hasOption("of")) {
			this.config.jCheckAutoFileName = true;
		}
		if (commandLine.hasOption("enc")) {
			this.config.setjComboEncType(commandLine.getOptionValue("enc"));
		}
		if (commandLine.hasOption("hoc")) {
			this.config.jRadioHorizontal = true;
			this.config.jRadioVertical = false;
		}
		if (commandLine.hasOption("d") && commandLine.hasOption("p")) {
			LogAppender.println("d・pオプションを併用するとpオプションが優先されます");
		}
		if (commandLine.hasOption("dev")) {
			this.config.loadProperties(this.presetPath+commandLine.getOptionValue("dev")+".ini");
		}
		if (commandLine.hasOption("p")) {
			this.config.loadProperties(commandLine.getOptionValue("p"));
		}




		try {
			//ePub出力クラス初期化
			this.epub3Writer = new Epub3Writer(this.jarPath+"template/");
			//ePub画像出力クラス初期化
			this.epub3ImageWriter = new Epub3ImageWriter(this.jarPath+"template/");

			//変換テーブルをstaticに生成
			this.aozoraConverter = new AozoraEpub3Converter(this.epub3Writer, this.jarPath);

		} catch (IOException e) {
			e.printStackTrace();
			LogAppender.append(e.getMessage());
		}

		String[] fileNames = commandLine.getArgs();

		try {

			Vector<File> vecFiles = new Vector<File>();
			//Web変換対象URLを格納
			Vector<String> vecUrlString = new Vector<String>();
			//ショートカットファイルを格納(同名の表紙取得に利用)
			Vector<File> vecUrlSrcFile = new Vector<File>();
			File dstPath = new File(commandLine.getOptionValue("d"));

			try {

				for (String str : fileNames) {
					if (str != null && (str.startsWith("http://") || str.startsWith("https://"))) {
						vecUrlString.add(str);
						vecUrlSrcFile.add(null);
					} else if (str.endsWith(".txt")) {
						File file = new File(str);
						if (file.isFile()) {
							if (dstPath == null && !isCacheFile(file)) dstPath = file.getParentFile();
							vecFiles.add(file);
						}
					}
				}
			} catch (Exception e) { e.printStackTrace(); }

			//何も変換しなければfalse
			if (vecFiles.size() == 0 && vecUrlString.size() == 0) return;
			//変換実行
			startConvertWorker(vecFiles, vecUrlString, vecUrlSrcFile, dstPath);
		} catch (Exception e) {
			e.printStackTrace();
		}





	}

	/** キャッシュパスを取得 */
	private File getCachePath()
	{
		String cachePathString = "";
		if("".equals(cachePathString)) cachePathString = this.jarPath+".cache";
		return new File(cachePathString);
	}
	/** キャッシュパスを以下のファイルならtrue */
	private boolean isCacheFile(File file)
	{
		try {
			return file.getCanonicalPath().startsWith(this.getCachePath().getCanonicalPath());
		} catch (IOException e) {
		}
		return false;
	}

	/** 別スレッド実行用SwingWorkerを実行
	 * @param dstPath 出力先 ブラウザからまたはURLペーストの場合はnull */
	void startConvertWorker(Vector<File> vecFiles, Vector<String> vecUrlString, Vector<File> vecUrlSrcFile, File dstPath)
	{
		//出力先が指定されていない場合は選択させる
		/*if (dstPath == null && jCheckSamePath.isSelected() || !jCheckSamePath.isSelected() && "".equals(jComboDstPath.getEditor().getItem().toString().trim())) {
			if (dstPath != null) this.currentPath = dstPath;
			dstPathChooser.actionPerformed(null);
			if (jCheckSamePath.isSelected() || "".equals(jComboDstPath.getEditor().getItem().toString().trim())) {
				LogAppender.println("変換処理を中止しました : "+(vecFiles.size()>0?vecFiles.get(0).getAbsoluteFile():vecUrlString.size()>0?vecUrlString.get(0):""));
				return;
			}
		}
		if (!jCheckSamePath.isSelected()) {
			dstPath = new File(jComboDstPath.getEditor().getItem().toString());
		}*/

		if (!dstPath.isDirectory()) {
			LogAppender.error("出力先がディレクトリではありません!! 変換処理を中止します");
			return;
		}

		//キャッシュパスを先にTextから取得
		this.cachePath = this.getCachePath();

		//web以下に同じ名前のパスがあったらキャッシュ後青空変換
		AE3SS.ConvertWorker convertWorker = new AE3SS.ConvertWorker(this, vecFiles, vecUrlString, vecUrlSrcFile, dstPath);
		convertWorker.execute();
	}

	public AE3SS getApplet() {
		return this;
	}


/////////////////////////////////////////////
	/** 複数ファイルを変換
	 * @param dstPath srcFileがキャッシュで入力ファイルを同じ場所に出力先指定をする場合 */
	private void convertFiles(Vector<File> vecSrcFiles, File dstPath)
	{
		File[] srcFiles = new File[vecSrcFiles.size()];
		for (int i=0; i<srcFiles.length; i++) {
			srcFiles[i] = vecSrcFiles.get(i);
		}
		this.convertFiles(srcFiles, dstPath);
	}
	/** 複数ファイルを変換
	 * @param dstPath srcFileがキャッシュで入力ファイルを同じ場所に出力先指定をする場合 */
	private void convertFiles(File[] srcFiles, File dstPath)
	{
		if (srcFiles.length == 0 ) return;

		convertCanceled = false;

		////////////////////////////////////////////////////////////////
		//Appletのパラメータを取得
		////////////////////////////////////////////////////////////////
		//画面サイズと画像リサイズ
		int resizeW = 0;
		if (this.config.jCheckResizeW) try { resizeW = Integer.parseInt(this.config.jTextResizeNumW); } catch (Exception e) {}
		int resizeH = 0;
		if (this.config.jCheckResizeH) try { resizeH = Integer.parseInt(this.config.jTextResizeNumH); } catch (Exception e) {}
		//int pixels = 0;
		//if (jCheckPixel.isSelected()) try { pixels = Integer.parseInt(jTextPixelW.getText())*Integer.parseInt(jTextPixelH.getText()); } catch (Exception e) {}
		int dispW = Integer.parseInt(this.config.jTextDispW);
		int dispH = Integer.parseInt(this.config.jTextDispH);
		this.coverW = Integer.parseInt(this.config.jTextCoverW);
		this.coverH = Integer.parseInt(this.config.jTextCoverH);
		int singlePageSizeW = Integer.parseInt(this.config.jTextSinglePageSizeW);
		int singlePageSizeH = Integer.parseInt(this.config.jTextSinglePageSizeH);
		int singlePageWidth = Integer.parseInt(this.config.jTextSinglePageWidth);

		float imageScale = 0;
		if (this.config.jCheckImageScale) try { imageScale = Float.parseFloat(this.config.jTextImageScale); } catch (Exception e) {}
		int imageFloatType = 0; //0=無効 1=上 2=下
		int imageFloatW = 0;
		int imageFloatH = 0;
		if (this.config.jCheckImageFloat) {
			imageFloatType = this.config.jComboImageFloatType+1;
			try { imageFloatW =Integer.parseInt(this.config.jTextImageFloatW); } catch (Exception e) {}
			try { imageFloatH =Integer.parseInt(this.config.jTextImageFloatH); } catch (Exception e) {}
		}
		float jpegQualty = 0.8f; try { jpegQualty = Integer.parseInt(this.config.jTextJpegQuality)/100f; } catch (Exception e) {}
		float gamma = 1.0f; if (this.config.jCheckGamma) try { gamma = Float.parseFloat(this.config.jTextGammaValue); } catch (Exception e) {}
		int autoMarginLimitH = 0;
		int autoMarginLimitV = 0;
		int autoMarginWhiteLevel = 0;
		float autoMarginPadding = 0;
		int autoMarginNombre = 0;
		float autoMarginNombreSize = 0.03f;
		if (this.config.jCheckAutoMargin) {
			try { autoMarginLimitH =Integer.parseInt(this.config.jTextAutoMarginLimitH); } catch (Exception e) {}
			try { autoMarginLimitV =Integer.parseInt(this.config.jTextAutoMarginLimitV); } catch (Exception e) {}
			try { autoMarginWhiteLevel =Integer.parseInt(this.config.jTextAutoMarginWhiteLevel); } catch (Exception e) {}
			try { autoMarginPadding =Float.parseFloat(this.config.jTextAutoMarginPadding); } catch (Exception e) {}
			autoMarginNombre = this.config.jComboAutoMarginNombre;
			try { autoMarginNombreSize =Float.parseFloat(this.config.jTextAutoMarginNombreSize)*0.01f; } catch (Exception e) {}
		}
		int rorateAngle = 0; if (this.config.jComboRotateImage == 1) rorateAngle = 90; else if (this.config.jComboRotateImage == 2) rorateAngle = -90;

		int imageSizeType = SectionInfo.IMAGE_SIZE_TYPE_ASPECT;
		if (this.config.jRadioImageSizeType1) imageSizeType = SectionInfo.IMAGE_SIZE_TYPE_AUTO;
		//else if (jRadioImageSizeType2.isSelected()) imageSizeType = SectionInfo.IMAGE_SIZE_TYPE_HEIGHT;

		//int imageFitType = SectionInfo.IMAGE_SIZE_TYPE_ASPECT;
		//if (jRadioImageFitType2.isSelected()) imageFitType = SectionInfo.IMAGE_SIZE_TYPE_HEIGHT;

		this.epub3Writer.setImageParam(dispW, dispH, coverW, coverH, resizeW, resizeH, singlePageSizeW, singlePageSizeH, singlePageWidth,
				imageSizeType, this.config.jCheckFitImage, this.config.jCheckSvgImage, rorateAngle,
				imageScale, imageFloatType, imageFloatW, imageFloatH, jpegQualty, gamma, autoMarginLimitH, autoMarginLimitV, autoMarginWhiteLevel, autoMarginPadding, autoMarginNombre, autoMarginNombreSize);
		this.epub3ImageWriter.setImageParam(dispW, dispH, coverW, coverH, resizeW, resizeH, singlePageSizeW, singlePageSizeH, singlePageWidth,
				imageSizeType, this.config.jCheckFitImage, this.config.jCheckSvgImage, rorateAngle,
				imageScale, imageFloatType, imageFloatW, imageFloatH, jpegQualty, gamma, autoMarginLimitH, autoMarginLimitV, autoMarginWhiteLevel, autoMarginPadding, autoMarginNombre, autoMarginNombreSize);
		//目次階層化設定
		this.epub3Writer.setTocParam(this.config.jCheckNavNest, this.config.jCheckNcxNest);

		//スタイル設定
		String[] pageMargin = new String[4];
		String pageMarginUnit = this.config.jRadioPageMarginUnit0?"em":"%";
		for (int i=0; i<pageMargin.length; i++) {
			pageMargin[i] = this.config.jTextPageMargins[i]+pageMarginUnit;
		}
		String[] bodyMargin = new String[4];
		String bodyMarginUnit = this.config.jRadioBodyMarginUnit0?"em":"%";
		for (int i=0; i<bodyMargin.length; i++) {
			bodyMargin[i] = this.config.jTextBodyMargins[i]+bodyMarginUnit;
		}
		float lineHeight = 1.8f;
		try { lineHeight = Float.parseFloat(this.config.jComboLineHeight); } catch (Exception e) {}
		int fontSize = 100;
		try { fontSize = (int)Float.parseFloat(this.config.jComboFontSize); } catch (Exception e) {}

		int dakutenType = this.config.jRadioDakutenType0 ? 0 : (this.config.jRadioDakutenType1 ? 1 : 2);

		this.epub3Writer.setStyles(pageMargin, bodyMargin, lineHeight, fontSize, this.config.jCheckBoldUseGothic, this.config.jCheckGothicUseBold);

		try {
			//挿絵なし
			this.aozoraConverter.setNoIllust(this.config.jCheckNoIllust);
			//栞用ID出力
			this.aozoraConverter.setWithMarkId(this.config.jCheckMarkId);
			//変換オプション設定
			this.aozoraConverter.setAutoYoko(this.config.jCheckAutoYoko, this.config.jCheckAutoYokoNum1, this.config.jCheckAutoYokoNum3, this.config.jCheckAutoEQ1);
			//文字出力設定
			this.aozoraConverter.setCharOutput(dakutenType, this.config.jCheckIvsBMP, this.config.jCheckIvsSSP);
			//全角スペースの禁則
			this.aozoraConverter.setSpaceHyphenation(this.config.jRadioSpaceHyp0?0:(this.config.jRadioSpaceHyp1?1:2));
			//注記のルビ表示
			this.aozoraConverter.setChukiRuby(this.config.jRadioChukiRuby1, this.config.jRadioChukiRuby2);
			//コメント
			this.aozoraConverter.setCommentPrint(this.config.jCheckCommentPrint, this.config.jCheckCommentConvert);

			//float表示
			this.aozoraConverter.setImageFloat(this.config.jCheckImageFloatPage, this.config.jCheckImageFloatBlock);

			//空行除去
			int removeEmptyLine = this.config.jComboxRemoveEmptyLine;
			int maxEmptyLine = this.config.jComboxMaxEmptyLine;
			this.aozoraConverter.setRemoveEmptyLine(removeEmptyLine, maxEmptyLine);

			//行頭字下げ
			this.aozoraConverter.setForceIndent(this.config.jCheckForceIndent);

			//強制改ページ
			if (this.config.jCheckPageBreak) {
				try {
					int forcePageBreakSize = 0;
					int forcePageBreakEmpty = 0;
					int forcePageBreakEmptySize = 0;
					int forcePageBreakChapter = 0;
					int forcePageBreakChapterSize = 0;
					forcePageBreakSize = Integer.parseInt(this.config.jTextPageBreakSize.trim()) * 1024;
					if (this.config.jCheckPageBreakEmpty) {
						forcePageBreakEmpty = Integer.parseInt(this.config.jComboxPageBreakEmptyLine);
						forcePageBreakEmptySize = Integer.parseInt(this.config.jTextPageBreakEmptySize.trim()) * 1024;
					} if (this.config.jCheckPageBreakChapter) {
						forcePageBreakChapter = 1;
						forcePageBreakChapterSize = Integer.parseInt(this.config.jTextPageBreakChapterSize.trim()) * 1024;
					}
					//Converterに設定
					this.aozoraConverter.setForcePageBreak(forcePageBreakSize, forcePageBreakEmpty, forcePageBreakEmptySize, forcePageBreakChapter, forcePageBreakChapterSize);
				} catch (Exception e) {
					LogAppender.println("強制改ページパラメータ読み込みエラー");
				}
			}

			//目次設定
			int maxLength = 64;
			try { maxLength = Integer.parseInt((this.config.jTextMaxChapterNameLength)); } catch (Exception e) {}

			this.aozoraConverter.setChapterLevel(maxLength, this.config.jCheckChapterExclude, this.config.jCheckChapterUseNextLine, this.config.jCheckChapterSection,
					this.config.jCheckChapterH, this.config.jCheckChapterH1, this.config.jCheckChapterH2, this.config.jCheckChapterH3, this.config.jCheckSameLineChapter,
					this.config.jCheckChapterName,
					this.config.jCheckChapterNumOnly, this.config.jCheckChapterNumTitle, this.config.jCheckChapterNumParen, this.config.jCheckChapterNumParenTitle,
					this.config.jCheckChapterPattern?this.config.jComboChapterPattern.trim():"");


			////////////////////////////////////////////////////////////////
			//すべてのファイルの変換実行
			////////////////////////////////////////////////////////////////

			this._convertFiles(srcFiles, dstPath);

			if (convertCanceled) {
				LogAppender.println("変換が中止されました");
			}

		} catch (Exception e) {
			e.printStackTrace();
			LogAppender.append("エラーが発生しました : ");
			LogAppender.println(e.getMessage());
		}
		////////////////////////////////
		System.gc();

	}
	/** サブディレクトリ再帰用 */
	private void _convertFiles(File[] srcFiles, File dstPath)
	{
		for (File srcFile : srcFiles) {
			if (srcFile.isDirectory()) {
				//サブディレクトリ 再帰
				_convertFiles(srcFile.listFiles(), dstPath);
			} else if (srcFile.isFile()) {
				convertFile(srcFile, dstPath);
			}
			//キャンセル
			if (convertCanceled) return;
		}
	}

	private void convertFile(File srcFile, File dstPath)
	{
		//拡張子
		String ext = srcFile.getName();
		ext = ext.substring(ext.lastIndexOf('.')+1).toLowerCase();


		//zipならzip内のテキストを検索
		int txtCount = 1;
		boolean imageOnly = false;
		LogAppender.append("------ ");
		if("zip".equals(ext) || "txtz".equals(ext)) {
			try {
				txtCount = AE3SS.countZipText(srcFile);
			} catch (Exception e) {
				e.printStackTrace();
			}
			if (txtCount == 0) { txtCount = 1; imageOnly = true; }
		} else if ("rar".equals(ext)) {
			try {
				txtCount = AE3SS.countRarText(srcFile);
			} catch (Exception e) {
				e.printStackTrace();
			}
			if (txtCount == 0) { txtCount = 1; imageOnly = true; }
		} else if ("cbz".equals(ext)) {
			imageOnly = true;
		} else if ("txt".equals(ext)) {
			LogAppender.println();
		}
		if (this.convertCanceled){
			LogAppender.println("変換処理を中止しました : "+srcFile.getAbsolutePath());
			return;
		}

		//キャッシュパスのファイルならエンコードを変換時のみUTF-8にする
		String encType = (String)this.config.jComboEncType;
		if (this.isCacheFile(srcFile)) this.config.setjComboEncType("UTF-8");
		try {
			for (int i=0; i<txtCount; i++) {
				convertFile(srcFile, dstPath, ext, i, imageOnly);
				if (convertCanceled) return;
			}
		} finally {
			//設定を戻す
			this.config.setjComboEncType(encType);
		}

	}
	/** 内部用変換関数 Appletの設定を引数に渡す
	 * @param srcFile 変換するファイル txt,zip,cbz,(rar,cbr)
	 * @param dstPath 出力先パス
	 * @param txtIdx Zip内テキストファイルの位置
	 */
	private void convertFile(File srcFile, File dstPath, String ext, int txtIdx, boolean imageOnly)
	{
		//パラメータ設定
		if (!"txt".equals(ext) && !"txtz".equals(ext) && !"zip".equals(ext) && !"cbz".equals(ext) && !"rar".equals(ext) ) {
			if (!"png".equals(ext) && !"jpg".equals(ext) && !"jpeg".equals(ext) && !"gif".equals(ext)) {
				LogAppender.println("txt, txtz, zip, cbz rar 以外は変換できません");
			}
			return;
		}
		//表紙にする挿絵の位置-1なら挿絵は使わない
		int coverImageIndex = -1;
		//表紙情報追加
		String coverFileName = this.config.jComboCover;
		if (coverFileName.equals(this.config.jComboCover_items[0])) {
			coverFileName = ""; //先頭の挿絵
			coverImageIndex = 0;
		} else if (coverFileName.equals(this.config.jComboCover_items[1])) {
			coverFileName = AE3SS.getSameCoverFileName(srcFile); //入力ファイルと同じ名前+.jpg/.png
		} else if (coverFileName.equals(this.config.jComboCover_items[2])) {
			coverFileName = null; //表紙無し
		}

		boolean isFile = "txt".equals(ext);
		ImageInfoReader imageInfoReader = new ImageInfoReader(isFile, srcFile);

		//zip内の画像をロード
		try {
			if (!isFile) {
				if ("rar".equals(ext)) {
					//Rar内の画像情報読み込み 画像のみならファイル順も格納
					imageInfoReader.loadRarImageInfos(srcFile, imageOnly);
				} else {
					//Zip内の画像情報読み込み 画像のみならファイル順も格納
					imageInfoReader.loadZipImageInfos(srcFile, imageOnly);
				}
			}
		} catch (Exception e) {
			e.printStackTrace();
			LogAppender.error(e.getMessage());
		}

		//BookInfo取得
		BookInfo bookInfo = null;
		try {
			if (!imageOnly) {
				//テキストファイルからメタ情報や画像単独ページ情報を取得
				bookInfo = AE3SS.getBookInfo(
						srcFile, ext, txtIdx, imageInfoReader, this.aozoraConverter,
						this.config.jComboEncType,
						BookInfo.TitleType.indexOf(this.config.jComboTitle),
						this.config.jCheckPubFirst
				);
			}
		} catch (Exception e) {
			LogAppender.error("ファイルが読み込めませんでした : "+srcFile.getPath());
			return;
		}

		if (this.convertCanceled){
			LogAppender.println("変換処理を中止しました : "+srcFile.getAbsolutePath());
			return;
		}

		Epub3Writer writer = this.epub3Writer;
		try {
			if (!isFile) {
				//Zip内の画像情報をbookInfoに設定
				if (imageOnly) {
					LogAppender.println("画像のみのePubファイルを生成します");
					//画像出力用のBookInfo生成
					bookInfo = new BookInfo(srcFile);
					bookInfo.imageOnly = true;
					//Writerを画像出力用派生クラスに入れ替え
					writer = this.epub3ImageWriter;

					if (imageInfoReader.countImageFileInfos() == 0) {
						LogAppender.error("画像がありませんでした");
						return;
					}

					//名前順で並び替え
					imageInfoReader.sortImageFileNames();
					//先頭画像をbookInfoに設定しておく
					//if (coverImageIndex == 0) {
					//	bookInfo.coverImage = imageInfoReader.getImage(0);
					//}

					//画像数をプログレスバーに設定 xhtml出力で+1 画像出力で+10
					//this.jProgressBar.setMaximum(imageInfoReader.countImageFileInfos()*11);
					//jProgressBar.setValue(0);
					//jProgressBar.setStringPainted(true);
				} else {
					//画像がなければプレビュー表示しないようにindexを-1に
					if (imageInfoReader.countImageFileNames() == 0) coverImageIndex = -1;

					//zipテキストならzip内の注記以外の画像も追加
					imageInfoReader.addNoNameImageFileName();
				}
			}
		} catch (Exception e) {
			e.printStackTrace();
			LogAppender.error(e.getMessage());
		}

		if (bookInfo == null) {
			LogAppender.error("書籍の情報が取得できませんでした");
			return;
		}

		//テキストなら行数/100と画像数をプログレスバーに設定
		if (bookInfo.totalLineNum > 0) {
			//if (isFile) this.jProgressBar.setMaximum(bookInfo.totalLineNum/10 + imageInfoReader.countImageFileNames()*10);
			//else this.jProgressBar.setMaximum(bookInfo.totalLineNum/10 + imageInfoReader.countImageFileInfos()*10);
			//jProgressBar.setValue(0);
			//jProgressBar.setStringPainted(true);
		}

		//表紙目次ページ出力設定
		bookInfo.insertCoverPage = this.config.jCheckCoverPage;
		bookInfo.insertTocPage = this.config.jCheckTocPage;
		bookInfo.insertCoverPageToc = this.config.jCheckCoverPageToc;
		bookInfo.insertTitleToc = this.config.jCheckTitleToc;
		//表題の見出しが非表示で行が追加されていたら削除
		if (!bookInfo.insertTitleToc && bookInfo.titleLine >= 0) {
			bookInfo.removeChapterLineInfo(bookInfo.titleLine);
		}

		//目次縦書き
		bookInfo.setTocVertical(this.config.jRadioTocV);
		//縦書き横書き設定追加
		bookInfo.vertical = this.config.jRadioVertical;
		this.aozoraConverter.vertical = bookInfo.vertical;

		//表紙設定
		//表題左右中央
		if (!this.config.jCheckTitlePage) {
			bookInfo.titlePageType = BookInfo.TITLE_NONE;
		} else if (this.config.jRadioTitleNormal) {
			bookInfo.titlePageType = BookInfo.TITLE_NORMAL;
		} else if (this.config.jRadioTitleMiddle) {
			bookInfo.titlePageType = BookInfo.TITLE_MIDDLE;
		} else if (this.config.jRadioTitleHorizontal) {
			bookInfo.titlePageType = BookInfo.TITLE_HORIZONTAL;
		}

		//先頭からの場合で指定行数以降なら表紙無し
		if ("".equals(coverFileName) && !imageOnly) {
			try {
				int maxCoverLine = Integer.parseInt(this.config.jTextMaxCoverLine);
				if (maxCoverLine > 0 && (bookInfo.firstImageLineNum == -1 || bookInfo.firstImageLineNum >= maxCoverLine)) {
					coverImageIndex = -1;
					coverFileName = null;
				} else {
					coverImageIndex = bookInfo.firstImageIdx;
				}
			} catch (Exception e) {}
		}

		//表紙ページの情報をbookInfoに設定
		bookInfo.coverFileName = coverFileName;
		bookInfo.coverImageIndex = coverImageIndex;

		String[] titleCreator = BookInfo.getFileTitleCreator(srcFile.getName());
		if (this.config.jCheckUseFileName) {
			//ファイル名優先ならテキスト側の情報は不要
			bookInfo.title = "";
			bookInfo.creator = "";
			if (titleCreator[0] != null) bookInfo.title = titleCreator[0];
			if (titleCreator[1] != null) bookInfo.creator = titleCreator[1];
		} else {
			//テキストから取得できなければファイル名を利用
			if (bookInfo.title == null || bookInfo.title.length() == 0) {
				bookInfo.title = titleCreator[0]==null?"":titleCreator[0];
				if (bookInfo.creator == null || bookInfo.creator.length() == 0) bookInfo.creator = titleCreator[1]==null?"":titleCreator[1];
			}
		}

		if (this.convertCanceled){
			LogAppender.println("変換処理を中止しました : "+srcFile.getAbsolutePath());
			return;
		}

		//前回の変換設定を反映
		BookInfoHistory history = this.getBookInfoHistory(bookInfo);
		if (history != null) {
			if (bookInfo.title.length() == 0) bookInfo.title = history.title;
			bookInfo.titleAs = history.titleAs;
			if (bookInfo.creator.length() == 0) bookInfo.creator = history.creator;
			bookInfo.creatorAs = history.creatorAs;
			if (bookInfo.publisher == null) bookInfo.publisher = history.publisher;
			//表紙設定
			if (this.config.jCheckCoverHistory) {
				bookInfo.coverEditInfo = history.coverEditInfo;
				bookInfo.coverFileName = history.coverFileName;
				bookInfo.coverExt = history.coverExt;
				bookInfo.coverImageIndex = history.coverImageIndex;

				//確認ダイアログ表示しない場合はイメージを生成
				//if (!this.config.jCheckConfirm && bookInfo.coverEditInfo != null) {
				//	try {
				//		this.jConfirmDialog.jCoverImagePanel.setBookInfo(bookInfo);
				//		if (bookInfo.coverImageIndex >= 0 && bookInfo.coverImageIndex < imageInfoReader.countImageFileNames()) {
				//			bookInfo.coverImage = imageInfoReader.getImage(bookInfo.coverImageIndex);
				//		} else if (bookInfo.coverImage == null && bookInfo.coverFileName != null) {
				//			bookInfo.loadCoverImage(bookInfo.coverFileName);
				//		}
				//		bookInfo.coverImage = this.jConfirmDialog.jCoverImagePanel.getModifiedImage(this.coverW, this.coverH);
				//	} catch (Exception e) { e.printStackTrace(); }
				//}
			}
		}

		String outExt = this.config.jComboExt.trim();
		////////////////////////////////
		//Kindleチェック
		File kindlegen = null;
		writer.setIsKindle(false);
		if (outExt.startsWith(".mobi")) {
			kindlegen = new File(this.jarPath+"kindlegen.exe");
			if (!kindlegen.isFile()) {
				kindlegen = new File(this.jarPath+"kindlegen");
				if (!kindlegen.isFile()) {
					kindlegen = null;
				}
			}
			if (kindlegen == null) {
				LogAppender.error("kindlegenがありません\nkindlegenをjarファイルの場所にコピーしてください");
				LogAppender.println("変換処理をキャンセルしました");
				return;
			}
			writer.setIsKindle(true);
		}

		//確認ダイアログ 変換ボタン押下時にbookInfo更新
		if (this.config.jCheckConfirm) {
		//	//表題と著者設定 ファイル名から設定
		//	String title = "";
		//	String creator = "";
		//	if (bookInfo.title != null) title = bookInfo.title;
		//	if (bookInfo.creator != null) creator = bookInfo.creator;
		//	this.jConfirmDialog.setChapterCheck(jCheckChapterSection.isSelected(),
		//			jCheckChapterH.isSelected(), jCheckChapterH1.isSelected(), jCheckChapterH2.isSelected(), jCheckChapterH3.isSelected(),
		//			jCheckChapterName.isSelected(),
		//			jCheckChapterNumOnly.isSelected()||jCheckChapterNumTitle.isSelected()||jCheckChapterNumParen.isSelected()||jCheckChapterNumParenTitle.isSelected(),
		//			jCheckChapterPattern.isSelected());
		//	this.jConfirmDialog.showDialog(
		//			srcFile,
		//			(dstPath!=null ? dstPath.getAbsolutePath() : srcFile.getParentFile().getAbsolutePath())+File.separator,
		//			title, creator, this.jComboTitle.getSelectedIndex(), jCheckPubFirst.isSelected(),
		//			bookInfo, imageInfoReader, this.jFrameParent.getLocation(),
		//			coverW, coverH
		//	);
		//	//ダイアログが閉じた後に再開
		//	if (this.jConfirmDialog.canceled) {
		//		this.convertCanceled = true;
		//		LogAppender.println("変換処理を中止しました : "+srcFile.getAbsolutePath());
		//		return;
		//	}
		//	if (this.jConfirmDialog.skipped) {
		//		this.setBookInfoHistory(bookInfo);
		//		LogAppender.println("変換をスキップしました : "+srcFile.getAbsolutePath());
		//		return;
		//	}
		//	//変換前確認のチェックを反映
		//	if (!this.jConfirmDialog.jCheckConfirm2.isSelected()) jCheckConfirm.setSelected(false);

		//	//確認ダイアログの値をBookInfoに設定
		//	bookInfo.title = this.jConfirmDialog.getMetaTitle();
		//	bookInfo.creator = this.jConfirmDialog.getMetaCreator();
		//	bookInfo.titleAs = this.jConfirmDialog.getMetaTitleAs();
		//	bookInfo.creatorAs = this.jConfirmDialog.getMetaCreatorAs();
		//	bookInfo.publisher = this.jConfirmDialog.getMetaPublisher();

		//	//著者が空欄なら著者行もクリア
		//	if (bookInfo.creator.length() == 0) bookInfo.creatorLine = -1;

			//プレビューでトリミングされていたらbookInfo.coverImageにBufferedImageを設定 それ以外はnullにする
		//	BufferedImage coverImage = this.jConfirmDialog.jCoverImagePanel.getModifiedImage(this.coverW, this.coverH);
		//	if (coverImage != null) {
		//		//Epub3Writerでイメージを出力
		//		bookInfo.coverImage = coverImage;
		//		//bookInfo.coverFileName = null;
		//		//元の表紙は残す
		//		if (this.jConfirmDialog.jCheckReplaceCover.isSelected()) bookInfo.coverImageIndex = -1;
		//	} else {
		//		bookInfo.coverImage = null;
		//	}

		//	this.setBookInfoHistory(bookInfo);
		} else {
			//表題の見出しが非表示で行が追加されていたら削除
			if (!bookInfo.insertTitleToc && bookInfo.titleLine >= 0) {
				bookInfo.removeChapterLineInfo(bookInfo.titleLine);
			}
		}

		boolean autoFileName = this.config.jCheckAutoFileName;
		boolean overWrite = this.config.jCheckOverWrite;

		//出力ファイル
		File outFile = null;

		//Kindleは一旦tmpファイルに出力
		File outFileOrg = null;
		if (kindlegen != null) {
			outFile = AE3SS.getOutFile(srcFile, dstPath, bookInfo, autoFileName, ".epub");
			File mobiFile = new File(outFile.getAbsolutePath().substring(0, outFile.getAbsolutePath().length()-4)+"mobi");
			if (!overWrite && (mobiFile.exists() || (outExt.endsWith(".epub") && outFile.exists()))) {
				LogAppender.println("変換中止: "+srcFile.getAbsolutePath());
				if (mobiFile.exists()) LogAppender.println("ファイルが存在します: "+mobiFile.getAbsolutePath());
				else LogAppender.println("ファイルが存在します: "+outFile.getAbsolutePath());
				return;
			}
			outFileOrg = outFile;
			try {
				outFile = File.createTempFile("kindle", ".epub", outFile.getParentFile());
				if (!outExt.endsWith(".epub")) outFile.deleteOnExit();
			} catch (IOException e) {
				outFile = outFileOrg;
				outFileOrg = null;
			}
		} else {
			outFile = AE3SS.getOutFile(srcFile, dstPath, bookInfo, autoFileName, outExt);
			//上書き確認
			if (!overWrite &&  outFile.exists()) {
				LogAppender.println("変換中止: "+srcFile.getAbsolutePath());
				LogAppender.println("ファイルが存在します: "+outFile.getAbsolutePath());
				return;
			}
		}
		/*
		if (overWrite &&  outFile.exists()) {
			int ret = JOptionPane.showConfirmDialog(this, "ファイルが存在します\n上書きしますか？\n(取り消しで変換キャンセル)", "上書き確認", JOptionPane.YES_NO_CANCEL_OPTION);
			if (ret == JOptionPane.NO_OPTION) {
				LogAppender.println("変換中止: "+srcFile.getAbsolutePath());
				return;
			} else if (ret == JOptionPane.CANCEL_OPTION) {
				LogAppender.println("変換中止: "+srcFile.getAbsolutePath());
				convertCanceled = true;
				LogAppender.println("変換処理をキャンセルしました");
				return;
			}
		}*/

		////////////////////////////////
		//変換実行
		AE3SS.convertFile(
				srcFile, ext, outFile,
				this.aozoraConverter,
				writer,
				this.config.jComboEncType,
				bookInfo, imageInfoReader, txtIdx
		);

		imageInfoReader = null;
		//画像は除去
		bookInfo.coverImage = null;

		//System.gc();

		//変換中にキャンセルされた場合
		if (this.convertCanceled) {
			LogAppender.println("変換処理を中止しました : "+srcFile.getAbsolutePath());
			return;
		}

		////////////////////////////////
		//kindlegen.exeがあれば実行
		try {
			if (kindlegen != null) {
				long time = System.currentTimeMillis();
				String outFileName = outFile.getAbsolutePath();
				LogAppender.println("kindlegenを実行します : "+kindlegen.getName()+" \""+outFileName+"\"");
				ProcessBuilder pb = new ProcessBuilder(kindlegen.getAbsolutePath(), "-locale", "en","-verbose", outFileName);
				this.kindleProcess = pb.start();
				BufferedReader br = new BufferedReader(new InputStreamReader(this.kindleProcess.getInputStream()));
				String line;
				int idx = 0;
				int cnt = 0;
				String msg = "";
				while ((line = br.readLine()) != null) {
					if (line.length() > 0) {
						System.out.println(line);
						if (msg.startsWith("Error")) msg += line;
						else msg = line;
						if (idx++ % 2 == 0) {
							if (cnt++ > 100) { cnt = 1; LogAppender.println(); }
							LogAppender.append(".");
						}
					}
				}
				br.close();
				if (convertCanceled) {
					LogAppender.println("\n"+msg+"\nkindlegenの変換を中断しました");
				} else {
					if (outFileOrg != null) {
						//mobiリネーム
						File mobiTmpFile = new File(outFile.getAbsolutePath().substring(0, outFile.getAbsolutePath().length()-4)+"mobi");
						File mobiFile = new File(outFileOrg.getAbsolutePath().substring(0, outFileOrg.getAbsolutePath().length()-4)+"mobi");
						if (mobiFile.exists()) mobiFile.delete();
						mobiTmpFile.renameTo(mobiFile);
						if (outExt.endsWith(".epub")) {
							//epubリネーム
							if (outFileOrg.exists()) outFileOrg.delete();
							outFile.renameTo(outFileOrg);
						} else {
							outFile.delete();
						}
						LogAppender.println("\n"+msg+"\nkindlegen変換完了 ["+(((System.currentTimeMillis()-time)/100)/10f)+"s] -> "+mobiFile.getName());
					}
				}
			}
		} catch (Exception e) {
			e.printStackTrace();
		} finally {
			if (this.kindleProcess != null) this.kindleProcess.destroy();
			this.kindleProcess = null;
		}
	}
	/** FILE END**/

	/** 別スレッド実行用SwingWorker */
	class ConvertWorker extends SwingWorker<Object, Object>
	{
		/** 面倒なのでbaseを渡す */
		AE3SS base;
		/** 変換対象ファイル */
		Vector<File> vecFiles;
		/** 変換対象URL */
		Vector<String> vecUrlString;
		/** ショートカットファイル */
		Vector<File> vecUrlSrcFile;

		File dstPath = null;

		/** @param dstPath ショートカットファイルなら同じ場所出力用に指定 */
		public ConvertWorker(AE3SS base, Vector<File> vecFiles, Vector<String> vecUrlString, Vector<File> vecUrlSrcFile, File dstPath)
		{

			this.vecFiles = vecFiles;
			this.vecUrlString = vecUrlString;
			this.vecUrlSrcFile = vecUrlSrcFile;
			this.base = base;

			this.dstPath = dstPath;
			try {

				//ファイルの変換
				if (this.vecFiles != null && vecFiles.size() >0) {
					this.base.convertFiles(vecFiles, dstPath);
				}
				if (vecUrlString != null && vecUrlString.size() >0) {
					this.base.convertWeb(vecUrlString, vecUrlSrcFile, dstPath);
				}
			} catch (Exception e) {
				e.printStackTrace();
				LogAppender.println("エラーが発生しました");
			}
		}

		@Override
		protected Object doInBackground() throws Exception
		{

			return null;
		}

		@Override
		protected void done()
		{
			super.done();
		}
	}

	/** 出力ファイルを生成 */
	static File getOutFile(File srcFile, File dstPath, BookInfo bookInfo, boolean autoFileName, String outExt)
	{
		//出力ファイル
		if (dstPath == null) dstPath = srcFile.getAbsoluteFile().getParentFile();
		String outFileName = "";
		if (autoFileName && (bookInfo.creator != null || bookInfo.title != null)) {
			outFileName = dstPath.getAbsolutePath()+"/";
			if (bookInfo.creator != null && bookInfo.creator.length() > 0) {
				String str = bookInfo.creator.replaceAll("[\\\\|\\/|\\:|\\*|\\?|\\<|\\>|\\||\\\"|\t]", "");
				if (str.length() > 64) str = str.substring(0, 64);
				outFileName += "["+str+"] ";
			}
			if (bookInfo.title != null) {
				outFileName += bookInfo.title.replaceAll("[\\\\|\\/|\\:|\\*|\\!|\\?|\\<|\\>|\\||\\\"|\t]", "");
			}
			if (outFileName.length() > 250) outFileName = outFileName.substring(0, 250);
		} else {
			outFileName = dstPath.getAbsolutePath()+"/"+srcFile.getName().replaceFirst("\\.[^\\.]+$", "");
		}
		if (outExt.length() == 0) outExt = ".epub";
		File outFile = new File(outFileName + outExt);
		//書き込み許可設定
		outFile.setWritable(true);

		return outFile;
	}

	/** 前処理で一度読み込んでタイトル等の情報を取得 */
	static public BookInfo getBookInfo(File srcFile, String ext, int txtIdx, ImageInfoReader imageInfoReader, AozoraEpub3Converter aozoraConverter,
			String encType, BookInfo.TitleType titleType, boolean pubFirst)
	{
		try {
			String[] textEntryName = new String[1];
			InputStream is = AE3SS.getTextInputStream(srcFile, ext, imageInfoReader, textEntryName, txtIdx);
			if (is == null) return null;

			//タイトル、画像注記、左右中央注記、目次取得
			BufferedReader src = new BufferedReader(new InputStreamReader(is, (String)encType));
			BookInfo bookInfo = aozoraConverter.getBookInfo(srcFile, src, imageInfoReader, titleType, pubFirst);
			is.close();
			bookInfo.textEntryName = textEntryName[0];
			return bookInfo;

		} catch (Exception e) {
			e.printStackTrace();
			LogAppender.append("エラーが発生しました : ");
			LogAppender.println(e.getMessage());
		}
		return null;
	}

	/** ファイルを変換
	 * @param srcFile 変換するファイル
	 * @param dstPath 出力先パス */
	static public void convertFile(File srcFile, String ext, File outFile, AozoraEpub3Converter aozoraConverter, Epub3Writer epubWriter,
			String encType, BookInfo bookInfo, ImageInfoReader imageInfoReader, int txtIdx)
	{
		try {
			long time = System.currentTimeMillis();
			LogAppender.append("変換開始 : ");
			LogAppender.println(srcFile.getPath());

			//入力Stream再オープン
			BufferedReader src = null;
			if (!bookInfo.imageOnly) {
				src = new BufferedReader(new InputStreamReader(getTextInputStream(srcFile, ext, null, null, txtIdx), encType));
			}

			//ePub書き出し srcは中でクローズされる
			epubWriter.write(aozoraConverter, src, srcFile, ext, outFile, bookInfo, imageInfoReader);

			LogAppender.append("変換完了["+(((System.currentTimeMillis()-time)/100)/10f)+"s] : ");
			LogAppender.println(outFile.getPath());

		} catch (Exception e) {
			e.printStackTrace();
			LogAppender.println("エラーが発生しました : "+e.getMessage());
			//LogAppender.printStaclTrace(e);
		}
	}

	/** 入力ファイルからStreamオープン
	 *
	 * @param srcFile
	 * @param ext
	 * @param imageInfoReader
	 * @param txtIdx テキストファイルのZip内の位置
	 * @return テキストファイルのストリーム (close()は呼び出し側ですること)
	 * @throws RarException
	 */
	static public InputStream getTextInputStream(File srcFile, String ext, ImageInfoReader imageInfoReader, String[] textEntryName, int txtIdx) throws IOException, RarException
	{
		if ("txt".equals(ext)) {
			return new FileInputStream(srcFile);
		} else if ("zip".equals(ext) || "txtz".equals(ext)) {
			//Zipなら最初のtxt
			ZipArchiveInputStream zis = new ZipArchiveInputStream(new BufferedInputStream(new FileInputStream(srcFile), 65536), "MS932", false);
			ArchiveEntry entry;
			while ((entry = zis.getNextEntry()) != null) {
				String entryName = entry.getName();
				if (entryName.substring(entryName.lastIndexOf('.')+1).equalsIgnoreCase("txt") && txtIdx-- == 0) {
					if (imageInfoReader != null) imageInfoReader.setArchiveTextEntry(entryName);
					if (textEntryName != null) textEntryName[0] = entryName;
					return zis;
				}
			}
			LogAppender.append("zip内にtxtファイルがありません: ");
			LogAppender.println(srcFile.getName());
			return null;
		} else if ("rar".equals(ext)) {
			//tempのtxtファイル作成
			Archive archive = new Archive(srcFile);
			try {
			FileHeader fileHeader = archive.nextFileHeader();
			while (fileHeader != null) {
				if (!fileHeader.isDirectory()) {
					String entryName = fileHeader.getFileNameW();
					if (entryName.length() == 0) entryName = fileHeader.getFileNameString();
					entryName = entryName.replace('\\', '/');
					if (entryName.substring(entryName.lastIndexOf('.')+1).equalsIgnoreCase("txt") && txtIdx-- == 0) {
						if (imageInfoReader != null) imageInfoReader.setArchiveTextEntry(entryName);
						if (textEntryName != null) textEntryName[0] = entryName;
						//tmpファイルにコピーして終了時に削除
						File tmpFile = File.createTempFile("rarTmp", "txt");
						tmpFile.deleteOnExit();
						FileOutputStream fos = new FileOutputStream(tmpFile);
						InputStream is = archive.getInputStream(fileHeader);
						try {
							IOUtils.copy(is, fos);
						} finally {
							is.close();
							fos.close();
						}
						return new BufferedInputStream(new FileInputStream(tmpFile), 65536);
					}
				}
				fileHeader = archive.nextFileHeader();
			}
			} finally {
				archive.close();
			}
			LogAppender.append("rar内にtxtファイルがありません: ");
			LogAppender.println(srcFile.getName());
			return null;
		} else {
			LogAppender.append("txt, zip, rar, txtz, cbz のみ変換可能です: ");
			LogAppender.println(srcFile.getPath());
		}
		return null;
	}

	/** Zipファイル内のテキストファイルの数を取得 */
	static public int countZipText(File zipFile) throws IOException
	{
		int txtCount = 0;
		ZipArchiveInputStream zis = new ZipArchiveInputStream(new BufferedInputStream(new FileInputStream(zipFile), 65536), "MS932", false);
		try {
			ArchiveEntry entry;
			while ((entry = zis.getNextEntry()) != null) {
				String entryName = entry.getName();
				if (entryName.substring(entryName.lastIndexOf('.')+1).equalsIgnoreCase("txt")) txtCount++;
			}
		} finally {
			zis.close();
		}
		return txtCount;
	}

	/** Ripファイル内のテキストファイルの数を取得 */
	static public int countRarText(File rarFile) throws IOException, RarException
	{
		int txtCount = 0;
		Archive archive = new Archive(rarFile);
		try {
			for (FileHeader fileHeader : archive.getFileHeaders()) {
				if (!fileHeader.isDirectory()) {
					String entryName = fileHeader.getFileNameW();
					if (entryName.length() == 0) entryName = fileHeader.getFileNameString();
					entryName = entryName.replace('\\', '/');
					if (entryName.substring(entryName.lastIndexOf('.')+1).equalsIgnoreCase("txt")) txtCount++;
				}
			}
		} finally {
			archive.close();
		}
		return txtCount;
	}

	/** 入力ファイルと同じ名前の画像を取得
	 * png, jpg, jpegの順で探す  */
	static public String getSameCoverFileName(File srcFile)
	{
		String baseFileName = srcFile.getPath();
		baseFileName = baseFileName.substring(0, baseFileName.lastIndexOf('.')+1);
		for (String ext : new String[]{"png","jpg","jpeg","PNG","JPG","JPEG","Png","Jpg","Jpeg"}) {
			String coverFileName = baseFileName+ext;
			if (new File(coverFileName).exists()) return coverFileName;
		}
		return null;
	}

	private void convertWeb(Vector<String> vecUrlString, Vector<File> vecUrlSrcFile, File dstPath) throws IOException
	{
		for (int i=0; i<vecUrlString.size(); i++) {
			String urlString = vecUrlString.get(i);
			File urSrcFile = null;
			if (vecUrlSrcFile != null && vecUrlSrcFile.size() > i) urSrcFile = vecUrlSrcFile.get(i);
			//URL変換 の最後が .zip .txtz .rar
			String ext = urlString.substring(urlString.lastIndexOf('.')+1).toLowerCase();
			if (ext.equals("zip") || ext.equals("txtz") || ext.equals("rar")) {

				String urlPath = urlString.substring(urlString.indexOf("//")+2).replaceAll("\\?\\*\\&\\|\\<\\>\"\\\\", "_");
				//青空zipのURLをキャッシュして変換
				//出力先 出力パスに保存
				File srcFile = new File(dstPath+"/"+new File(urlPath).getName());
				LogAppender.println("出力先にダウンロードします : "+srcFile.getCanonicalPath());
				srcFile.getParentFile().mkdirs();
				//ダウンロード
				BufferedInputStream bis = new BufferedInputStream(new URL(urlString).openStream(), 8192);
				BufferedOutputStream bos = new BufferedOutputStream(new FileOutputStream(srcFile));
				IOUtils.copy(bis, bos);
				bos.close();
				bis.close();

				//変換実行
				this.convertFiles(new File[]{srcFile}, dstPath);

				continue;
			}

			try {
				LogAppender.println("--------");
				LogAppender.append(urlString);
				LogAppender.println(" を読み込みます");

				webConverter = WebAozoraConverter.createWebAozoraConverter(urlString, webConfigPath);
				if (webConverter == null) {
					LogAppender.append(urlString);
					LogAppender.println(" は変換できませんでした");
					continue;
				}

				int interval = 500;
				try { interval = (int)(Float.parseFloat(this.config.jTextWebInterval)*1000); } catch (Exception e) {}
				int beforeChapter = 0;
				if (this.config.jCheckWebBeforeChapter) {
					try { beforeChapter = Integer.parseInt(this.config.jTextWebBeforeChapterCount); } catch (Exception e) {}
				}
				float modifiedExpire = 0;
				try { modifiedExpire = Float.parseFloat(this.config.jTextWebModifiedExpire); } catch (Exception e) {}
				//キャッシュパス
				if (!this.cachePath.isDirectory()) {
					this.cachePath.mkdirs();
					LogAppender.println("キャッシュパスを作成します : "+this.cachePath.getCanonicalPath());
				}
				if (!this.cachePath.isDirectory()) {
					LogAppender.println("キャッシュパスが作成できませんでした");
					return;
				}

				File srcFile = webConverter.convertToAozoraText(urlString, this.cachePath, interval, modifiedExpire,
						this.config.jCheckWebConvertUpdated, this.config.jCheckWebModifiedOnly, this.config.jCheckWebModifiedTail,
						beforeChapter);

				if (srcFile == null) {
					LogAppender.append(urlString);
					if (this.config.jCheckWebConvertUpdated && !webConverter.isUpdated()
							|| this.config.jCheckWebModifiedOnly && !webConverter.isUpdated())
						LogAppender.println(" の変換をスキップしました");
					else if (webConverter.isCanceled())
						LogAppender.println(" の変換をキャンセルしました");
					else
						LogAppender.println(" は変換できませんでした");
					continue;
				}

				//エンコードを変換時のみUTF-8にする
				String encType = (String)this.config.jComboEncType;
				this.config.setjComboEncType("UTF-8");
				int titleTypeIdx = this.config.jComboTitle;
				this.config.setjComboTitle(0);
				boolean checkUseFileName = this.config.jCheckUseFileName;
				this.config.setjCheckUseFileName(false);
				//コメント出力
				boolean commentPrint = this.config.jCheckCommentPrint;
				this.config.setjCheckCommentPrint(true);
				boolean commentConvert = this.config.jCheckCommentConvert;
				this.config.setjCheckCommentConvert(true);

				//表紙画像はconverted.pngで保存される 指定がない場合はそれを利用する
				String coverItem = this.config.jComboCover;
				//入力ファイルと同じ表紙の指定の場合 ショートカットファイルのパスにファイルがあればファイルパスを指定に変更
				if (this.config.jComboCover_index == 1 && urSrcFile != null) {
					String coverFileName = AE3SS.getSameCoverFileName(urSrcFile);
					this.config.setjComboCover(coverFileName);
					this.config.setjComboCover(-1);
				}
				//同名のファイルが無い場合はconverted.pngを利用する設定に変更
				if (this.config.jComboCover_index == 0 || this.config.jComboCover_index== 1) {
					this.config.setjComboCover(1);
				}

				//変換処理実行
				convertFiles(new File[]{srcFile}, dstPath);

				//設定を戻す
				this.config.setjComboEncType(encType);
				this.config.setjComboTitle(titleTypeIdx);
				this.config.setjCheckUseFileName(checkUseFileName);
				this.config.setjCheckCommentPrint(commentPrint);
				this.config.setjCheckCommentConvert(commentConvert);
				this.config.setjComboCover(coverItem);

			} catch (Exception e) {
				e.printStackTrace(); LogAppender.println("エラーが発生しました : "+e.getMessage());
			}
		}
	}

	LinkedHashMap<String, BookInfoHistory> mapBookInfoHistory = new LinkedHashMap<String, BookInfoHistory>(){
		private static final long serialVersionUID = 1L;
		@SuppressWarnings("rawtypes")
		protected boolean removeEldestEntry(Map.Entry eldest) { return size() > 256; }
	};

	//以前の変換情報取得
	BookInfoHistory getBookInfoHistory(BookInfo bookInfo)
	{
		String key = bookInfo.srcFile.getAbsolutePath();
		if (bookInfo.textEntryName != null) key += "/"+bookInfo.textEntryName;
		return mapBookInfoHistory.get(key);
	}

	void setBookInfoHistory(BookInfo bookInfo)
	{
		String key = bookInfo.srcFile.getAbsolutePath();
		if (bookInfo.textEntryName != null) key += "/"+bookInfo.textEntryName;
		mapBookInfoHistory.put(key, new BookInfoHistory(bookInfo));
	}

	class Config {
		AE3SS main;

		boolean jCheckResizeW = false;
		String jTextResizeNumW = "2048";
		boolean jCheckResizeH = false;
		String jTextResizeNumH = "2048";
		String jTextDispW = "600";
		String jTextDispH = "800";
		String jTextCoverW = "600";
		String jTextCoverH = "800";
		String jTextSinglePageSizeW = "400";
		String jTextSinglePageSizeH = "600";
		String jTextSinglePageWidth = "600";
		boolean jCheckImageScale = false;
		String jTextImageScale = "1.0";
		boolean jCheckImageFloat = false;
		String jTextImageFloatW = "600";
		String jTextImageFloatH = "400";
		int jComboImageFloatType = -1;
		boolean jCheckAutoMargin = false;
		String jTextJpegQuality = "85";
		boolean jCheckGamma = false;
		String jTextGammaValue = "1.0";
		String jTextAutoMarginLimitH = "15";
		String jTextAutoMarginLimitV = "15";
		String jTextAutoMarginPadding = "1.0";
		String jTextAutoMarginWhiteLevel = "80";
		int jComboAutoMarginNombre = -1;
		String jTextAutoMarginNombreSize = "3.0";
		int jComboRotateImage = -1;
		boolean jRadioImageSizeType1 = true;
		boolean jCheckFitImage = false;
		boolean jCheckSvgImage = false;
		boolean jCheckNavNest = false;
		boolean jCheckNcxNest = false;
		boolean jRadioPageMarginUnit0 = true;
		boolean jRadioPageMarginUnit1 = false;
		String[] jTextPageMargins = {"0.5", "0.5", "0.5", "0.5"};
		boolean jRadioBodyMarginUnit0 = true;
		boolean jRadioBodyMarginUnit1 = false;
		String[] jTextBodyMargins = {"0", "0", "0", "0"};
		String jComboLineHeight = "1.8"; // {"1.3", "1.4", "1.5", "1.6", "1.7", "1.8", "1.9", "2.0"}
		String jComboFontSize = "100"; // {"75", "80", "85", "90", "95", "100", "105", "110", "115", "120", "125"}
		boolean jRadioDakutenType0 = false;
		boolean jRadioDakutenType1 = false;
		boolean jRadioDakutenType2 = true;
		boolean jCheckBoldUseGothic = false;
		boolean jCheckGothicUseBold = false;
		boolean jCheckNoIllust = false;
		boolean jCheckMarkId = false;
		boolean jCheckAutoYoko = true;
		boolean jCheckAutoYokoNum1 = false;
		boolean jCheckAutoYokoNum3 = false;
		boolean jCheckAutoEQ1 = false;
		boolean jCheckCommentPrint = false;
		boolean jCheckCommentConvert = false;
		boolean jCheckIvsBMP = false;
		boolean jCheckIvsSSP = false;
		boolean jRadioSpaceHyp0 = true;
		boolean jRadioSpaceHyp1 = false;
		boolean jRadioSpaceHyp2 = false;
		boolean jRadioChukiRuby0 = true;
		boolean jRadioChukiRuby1 = false;
		boolean jRadioChukiRuby2 = false;
		boolean jCheckImageFloatPage = false;
		boolean jCheckImageFloatBlock = false;
		int jComboxRemoveEmptyLine = 0; //{"0", "1", "2", "3", "4", "5"}
		int jComboxMaxEmptyLine = -1; // {"-", "1", "2", "3", "4", "5", "6", "7", "8", "9", "10"}
		boolean jCheckForceIndent = false;
		boolean jCheckPageBreak = true;
		String jTextPageBreakSize = "400";
		boolean jCheckPageBreakEmpty = false;
		String jComboxPageBreakEmptyLine = "2"; // {"1", "2", "3", "4", "5", "6", "7", "8", "9"}
		String[] jComboxPageBreakEmptyLine_items = {"1", "2", "3", "4", "5", "6", "7", "8", "9"};
		String jTextPageBreakEmptySize = "300";
		boolean jCheckPageBreakChapter = false;
		String jTextPageBreakChapterSize = "200";
		String jTextMaxChapterNameLength = "64";
		boolean jCheckCoverPageToc = false;
		boolean jCheckTitleToc = true;
		boolean jCheckChapterUseNextLine = false;
		boolean jCheckChapterExclude = true;
		boolean jCheckChapterH = true;
		boolean jCheckChapterH1 = true;
		boolean jCheckChapterH2 = true;
		boolean jCheckChapterH3 = true;
		boolean jCheckSameLineChapter = false;
		boolean jCheckChapterSection = true;
		boolean jCheckChapterName = true;
		boolean jCheckChapterNumOnly = false;
		boolean jCheckChapterNumTitle = false;
		boolean jCheckChapterNumParen = false;
		boolean jCheckChapterNumParenTitle = false;
		boolean jCheckChapterPattern = false;
		String jComboChapterPattern = "^(見出し１|見出し２|見出し３)$"; // {"^(見出し１|見出し２|見出し３)$", "^(†|【|●|▼|■)", "^(0-9|０-９|一|二|三|四|五|六|七|八|九|十|〇)", "^[1|2|１|２]?[0-9|０-９]月[1-3|１-３]?[0-9|０-９]日", "^(一|十)?(一|二|三|四|五|六|七|八|九|十|〇)月(一|十|二十?|三十?)?(一|二|三|四|五|六|七|八|九|十|〇)日"}
		String[] jComboChapterPattern_items = {"^(見出し１|見出し２|見出し３)$", "^(†|【|●|▼|■)", "^(0-9|０-９|一|二|三|四|五|六|七|八|九|十|〇)", "^[1|2|１|２]?[0-9|０-９]月[1-3|１-３]?[0-9|０-９]日", "^(一|十)?(一|二|三|四|五|六|七|八|九|十|〇)月(一|十|二十?|三十?)?(一|二|三|四|五|六|七|八|九|十|〇)日"};
		String jComboEncType = "MS932";
		String[] jComboEncType_items = {"MS932", "UTF-8"};
		String jComboCover = "[先頭の挿絵]";//{"[先頭の挿絵]", "[入力ファイル名と同じ画像(png,jpg)]", "[表紙無し]", "http://"}
		int jComboCover_index = 0;
		String[] jComboCover_items = {"[先頭の挿絵]", "[入力ファイル名と同じ画像(png,jpg)]", "[表紙無し]", "http://"};
		int jComboTitle = 0;//{"表題 → 著者名", "著者名 → 表題", "表題 → 著者名(副題優先)", "表題のみ(1行)", "表題+著者のみ(2行)", "なし"}
		boolean jCheckPubFirst = false;
		boolean jCheckCoverPage = true;
		boolean jCheckTitlePage = true;
		boolean jRadioTitleNormal = false;
		boolean jRadioTitleMiddle = true;
		boolean jRadioTitleHorizontal = false;
		boolean jCheckTocPage = false;
		boolean jRadioTocV = true;
		boolean jRadioTocH = false;
		boolean jRadioVertical = true;
		boolean jRadioHorizontal = false;
		String jTextMaxCoverLine = "10";
		boolean jCheckUseFileName = false;
		boolean jCheckCoverHistory = true;
		boolean jCheckConfirm = false; // !!important??
		String jComboExt = ".epub";//{".epub", ".kepub.epub", ".fxl.kepub.epub", ".mobi", ".mobi+.epub"}
		boolean jCheckAutoFileName = true;
		boolean jCheckOverWrite = true;
		boolean jCheckSamePath = true;
		String jComboDstPath = "";
		boolean jRadioImageSizeType3 = false;

		String jTextWebInterval = "0.5";
		String jTextCachePath = ".cache";
		boolean jCheckWebConvertUpdated = false;
		boolean jCheckWebBeforeChapter = false;
		String jTextWebBeforeChapterCount = "1";
		boolean jCheckWebModifiedOnly = false;
		boolean jCheckWebModifiedTail = false;
		String jTextWebModifiedExpire = "24";


		public Config(AE3SS main) {
			this.main = main;
		}

		public void setjComboEncType(String EncType) {
			this.jComboEncType = EncType;
		}

		public void setjComboEncType(int idx) {
			this.jComboEncType = this.jComboEncType_items[idx];
		}

		public void setjComboTitle(int id) {
			this.jComboTitle = id;
		}

		public void setjCheckUseFileName(boolean bool) {
			this.jCheckUseFileName = bool;
		}

		public void setjCheckCommentPrint(boolean bool) {
			this.jCheckCommentPrint = bool;
		}

		public void setjCheckCommentConvert(boolean bool) {
			this.jCheckCommentConvert = bool;
		}

		public void setjComboCover(String path) {
			this.jComboCover = path;
			this.jComboCover_index = -1;
		}

		public void setjComboCover(int idx) {
			this.jComboCover_index = idx;
			this.jComboCover = this.jComboCover_items[idx];
		}


		private boolean setPropsSelected(boolean bool, Properties props, String name) {
			if (props.containsKey(name)) {
				bool = "1".equals(props.getProperty(name));
			} else {
				return bool;
			}
			return bool;
		}

		private boolean setPropsSelected(boolean bool, Properties props, String name, boolean nullSelect) {
			if (props.containsKey(name)) {
				bool = "1".equals(props.getProperty(name));
			} else {
				bool = nullSelect;
			}
			return bool;
		}

		/** テキスト値を設定 null なら設定しない */
		private void setPropsText(String text, Properties props, String name) {
			try {
				if (!props.containsKey(name)) return;
				text = props.getProperty(name);
			} catch (Exception e) {}
		}

		/** int値を設定 null なら設定しない */
		private void setPropsIntText(String text, Properties props, String name) {
			try {
				if (!props.containsKey(name)) return;
				text = Integer.toString(Integer.parseInt(props.getProperty(name)));
			} catch (Exception e) {}
		}

		/** float値を設定 null なら設定しない */
		private void setPropsFloatText(String text, Properties props, String name) {
			try {
				if (!props.containsKey(name)) return;
				text = Float.toString(Float.parseFloat(props.getProperty(name)));
			} catch (Exception e) {}
		}

		/** 数値を設定 null なら設定しない */
		private void setPropsNumberText(String text, Properties props, String name) {
			try {
				if (!props.containsKey(name)) return;
				text = NumberFormat.getNumberInstance().format(Float.parseFloat(props.getProperty(name)));
			} catch (Exception e) {}
		}


		public void loadProperties(String path) {
			props = new Properties();
			try {
				FileInputStream fos = new FileInputStream(path);
				props.load(fos);
				fos.close();
			} catch (Exception e) { }
			this.loadProperties(props);
		}

		public void loadProperties(Properties props) {
			boolean selected;

			//表題
			try { jComboTitle = Integer.parseInt(props.getProperty("TitleType")); } catch (Exception e) {}
			setPropsSelected(jCheckPubFirst, props, "PubFirst");
			setPropsSelected(jCheckUseFileName, props, "UseFileName");
			//表紙
			if (props.getProperty("Cover")==null||props.getProperty("Cover").length()==0) setjComboCover(0);
			else setjComboCover(props.getProperty("Cover"));
			//表紙履歴
			setPropsSelected(jCheckCoverHistory, props, "CoverHistory");
			//有効行数
			setPropsIntText(jTextMaxCoverLine, props, "MaxCoverLine");

			setPropsSelected(jCheckCoverPage, props, "CoverPage");
			//表題ページ
			setPropsSelected(jCheckTitlePage, props, "TitlePageWrite");
			String propValue = props.getProperty("TitlePage");
			if (propValue != null) {
				jRadioTitleNormal = Integer.toString(BookInfo.TITLE_NORMAL).equals(propValue);
				jRadioTitleMiddle = Integer.toString(BookInfo.TITLE_MIDDLE).equals(propValue);
				jRadioTitleHorizontal = Integer.toString(BookInfo.TITLE_HORIZONTAL).equals(propValue);
			}

			setPropsSelected(jCheckTocPage, props, "TocPage");
			selected = setPropsSelected(jRadioTocV, props, "TocVertical");
			jRadioTocH = !selected;
			//出力ファイル設定
			//拡張子
			if (props.getProperty("Ext") != null && props.getProperty("Ext").length() > 0)
				jComboExt = props.getProperty("Ext");
			//出力ファイル名設定
			setPropsSelected(jCheckAutoFileName, props, "AutoFileName");
			//ファイルの上書き許可
			setPropsSelected(jCheckOverWrite, props, "OverWrite");
			//栞用ID出力
			setPropsSelected(jCheckMarkId, props, "MarkId");
			//4バイト文字を変換する
			//setPropsSelected(jCheckGaiji32, props, "Gaiji32");
			//縦書き横書き
			selected= setPropsSelected(jRadioVertical, props, "Vertical");
			jRadioHorizontal = !selected;
			//入力文字コード
			try { setjComboEncType(Integer.parseInt(props.getProperty("EncType"))); } catch (Exception e) {}

			////////////////////////////////////////////////////////////////
			//画像設定
			setPropsSelected(jCheckNoIllust, props, "NoIllust");
			//画面サイズ
			setPropsIntText(jTextDispW, props, "DispW");
			setPropsIntText(jTextDispH, props, "DispH");
			//表紙サイズ
			setPropsIntText(jTextCoverW, props, "CoverW");
			setPropsIntText(jTextCoverH, props, "CoverH");
			//画像単ページ化
			setPropsIntText(jTextSinglePageSizeW, props, "SinglePageSizeW");
			setPropsIntText(jTextSinglePageSizeH, props, "SinglePageSizeH");
			//横のみ
			setPropsIntText(jTextSinglePageWidth, props, "SinglePageWidth");
			//サイズ指定
			propValue = props.getProperty("ImageSizeType");
			if (propValue != null) {
				jRadioImageSizeType1 = Integer.toString(SectionInfo.IMAGE_SIZE_TYPE_AUTO).equals(propValue);
				jRadioImageSizeType3 = Integer.toString(SectionInfo.IMAGE_SIZE_TYPE_ASPECT).equals(propValue);
			}
			//拡大しない
			setPropsSelected(jCheckFitImage, props, "FitImage");
			//SVG画像タグ出力
			setPropsSelected(jCheckSvgImage, props, "SvgImage");
			try { jComboRotateImage = Integer.parseInt(props.getProperty("RotateImage")); } catch (Exception e) {}
			//画像倍率
			setPropsSelected(jCheckImageScale, props, "ImageScaleChecked", false);
			setPropsFloatText(jTextImageScale, props, "ImageScale");
			//画像回り込み
			setPropsSelected(jCheckImageFloat, props, "ImageFloat");
			setPropsIntText(jTextImageFloatW, props, "ImageFloatW");
			setPropsIntText(jTextImageFloatH, props, "ImageFloatH");
			try { jComboImageFloatType = Integer.parseInt(props.getProperty("ImageFloatType")); } catch (Exception e) {}
			//画像縮小指定
			setPropsSelected(jCheckResizeW, props, "ResizeW");
			setPropsIntText(jTextResizeNumW, props, "ResizeNumW");
			setPropsSelected(jCheckResizeH, props, "ResizeH");
			setPropsIntText(jTextResizeNumH, props, "ResizeNumH");
			//Float表示 (デフォルトOFF)
			setPropsSelected(jCheckImageFloatPage, props, "ImageFloatPage", false);
			setPropsSelected(jCheckImageFloatBlock, props, "ImageFloatBlock", false);
			//Jpeg圧縮率
			setPropsIntText(jTextJpegQuality, props, "JpegQuality");
			//ガンマ補正
			setPropsSelected(jCheckGamma, props, "Gamma");
			setPropsFloatText(jTextGammaValue, props, "GammaValue");
			//余白除去
			setPropsSelected(jCheckAutoMargin, props, "AutoMargin");
			setPropsIntText(jTextAutoMarginLimitH, props, "AutoMarginLimitH");
			setPropsIntText(jTextAutoMarginLimitV, props, "AutoMarginLimitV");
			setPropsIntText(jTextAutoMarginWhiteLevel, props, "AutoMarginWhiteLevel");
			setPropsFloatText(jTextAutoMarginPadding, props, "AutoMarginPadding");
			try { jComboAutoMarginNombre = Integer.parseInt(props.getProperty("AutoMarginNombre")); } catch (Exception e) {}
			setPropsFloatText(jTextAutoMarginNombreSize, props, "AutoMarginNombreSize");

			////////////////////////////////////////////////////////////////
			//詳細設定
			propValue = props.getProperty("SpaceHyphenation");
			if (propValue != null) {
				jRadioSpaceHyp1 = "1".equals(propValue);
				jRadioSpaceHyp2 = "2".equals(propValue);
				jRadioSpaceHyp0 = "0".equals(propValue);
			}
			//注記のルビ表示
			propValue = props.getProperty("ChukiRuby");
			if (propValue != null) {
				jRadioChukiRuby0 = "0".equals(propValue);
				jRadioChukiRuby1 = "1".equals(propValue);
				jRadioChukiRuby2 = "2".equals(propValue);
			}
			//自動縦中横
			//半角2文字縦書き
			setPropsSelected(jCheckAutoYoko, props, "AutoYoko");
			//半角数字1文字縦書き
			setPropsSelected(jCheckAutoYokoNum1, props, "AutoYokoNum1");
			//半角数字3文字縦書き
			setPropsSelected(jCheckAutoYokoNum3, props, "AutoYokoNum3");
			//!? 1文字
			setPropsSelected(jCheckAutoEQ1, props, "AutoYokoEQ1");
			//コメント出力
			setPropsSelected(jCheckCommentPrint, props, "CommentPrint");
			setPropsSelected(jCheckCommentConvert, props, "CommentConvert");
			//空行除去
			try { jComboxRemoveEmptyLine = Integer.parseInt(props.getProperty("RemoveEmptyLine")); } catch (Exception e) {}
			propValue = props.getProperty("MaxEmptyLine");
			try { jComboxMaxEmptyLine = Integer.parseInt(propValue); } catch (Exception e) {}
			//行頭字下げ追加
			setPropsSelected(jCheckForceIndent, props, "ForceIndent");
			//強制改ページ
			setPropsSelected(jCheckPageBreak, props, "PageBreak");
			try { jTextPageBreakSize = Integer.toString(Integer.parseInt(props.getProperty("PageBreakSize"))); } catch (Exception e) {}
			setPropsSelected(jCheckPageBreakEmpty, props, "PageBreakEmpty");
			propValue = props.getProperty("PageBreakEmptyLine");
			if (propValue != null) jComboxPageBreakEmptyLine = propValue;
			setPropsIntText(jTextPageBreakEmptySize, props, "PageBreakEmptySize");
			setPropsSelected(jCheckPageBreakChapter, props, "PageBreakChapter");
			setPropsIntText(jTextPageBreakChapterSize, props, "PageBreakChapterSize");

			////////////////////////////////////////////////////////////////
			//目次設定
			//最大文字数
			setPropsIntText(jTextMaxChapterNameLength, props, "MaxChapterNameLength");
			//表紙
			setPropsSelected(jCheckCoverPageToc, props, "CoverPageToc");
			setPropsSelected(jCheckTitleToc, props, "TitleToc");
			setPropsSelected(jCheckChapterUseNextLine, props, "ChapterUseNextLine");
			setPropsSelected(jCheckChapterExclude, props, "ChapterExclude");
			//目次階層化
			setPropsSelected(jCheckNavNest, props, "NavNest");
			setPropsSelected(jCheckNcxNest, props, "NcxNest");
			//改ページ後を目次に追加
			setPropsSelected(jCheckChapterSection, props, "ChapterSection");
			//見出し注記
			setPropsSelected(jCheckChapterH, props, "ChapterH");
			setPropsSelected(jCheckChapterH1, props, "ChapterH1");
			setPropsSelected(jCheckChapterH2, props, "ChapterH2");
			setPropsSelected(jCheckChapterH3, props, "ChapterH3");
			setPropsSelected(jCheckSameLineChapter, props, "SameLineChapter");
			//章番号、数字、パターン
			setPropsSelected(jCheckChapterName, props, "ChapterName");
			setPropsSelected(jCheckChapterNumOnly, props, "ChapterNumOnly");
			setPropsSelected(jCheckChapterNumTitle, props, "ChapterNumTitle");
			setPropsSelected(jCheckChapterNumParen, props, "ChapterNumParen");
			setPropsSelected(jCheckChapterNumParenTitle, props, "ChapterNumParenTitle");
			setPropsSelected(jCheckChapterPattern, props, "ChapterPattern");
			if (props.containsKey("ChapterPatternText")) jComboChapterPattern = props.getProperty("ChapterPatternText");

			////////////////////////////////////////////////////////////////
			//スタイル
			propValue = props.getProperty("PageMargin");
			if (propValue != null) {
				String[] pageMargins = propValue.split(",");
				for (int i=0; i<pageMargins.length; i++) jTextPageMargins[i] = pageMargins[i];
			}
			propValue = props.getProperty("PageMarginUnit");
			if (propValue != null) {
				jRadioPageMarginUnit0 = "0".equals(propValue);
				jRadioPageMarginUnit1 = "1".equals(propValue);
				//jRadioPageMarginUnit2.setSelected("2".equals(propValue));
			}
			propValue = props.getProperty("BodyMargin");
			if (propValue != null) {
				String[] bodyMargins = propValue.split(",");
				for (int i=0; i<bodyMargins.length; i++) jTextBodyMargins[i] = bodyMargins[i];
			}
			propValue = props.getProperty("BodyMarginUnit");
			if (propValue != null) {
				jRadioBodyMarginUnit0 = "0".equals(propValue);
				jRadioBodyMarginUnit1 = "1".equals(propValue);
				//jRadioBodyMarginUnit2.setSelected("2".equals(propValue));
			}
			propValue = props.getProperty("LineHeight");
			if (propValue != null && !"".equals(propValue)) jComboLineHeight = propValue;
			propValue = props.getProperty("FontSize");
			if (propValue != null && !"".equals(propValue)) jComboFontSize = propValue;
			setPropsSelected(jCheckBoldUseGothic, props, "BoldUseGothic");
			setPropsSelected(jCheckGothicUseBold, props, "GothicUseBold");

			//文字
			propValue = props.getProperty("DakutenType");
			if (propValue != null) {
				jRadioDakutenType0 = "0".equals(propValue);
				jRadioDakutenType1 = "1".equals(propValue);
				jRadioDakutenType2 = "2".equals(propValue);
			}
			setPropsSelected(jCheckIvsBMP, props, "IvsBMP");
			setPropsSelected(jCheckIvsSSP, props, "IvsSSP");

			////////////////////////////////////////////////////////////////
			//Web
			setPropsFloatText(jTextWebInterval, props, "WebInterval");
			setPropsText(jTextCachePath, props, "CachePath");
			if ("".equals(jTextCachePath)) jTextCachePath = ".cache";
			setPropsNumberText(jTextWebModifiedExpire, props, "WebModifiedExpire");
			setPropsSelected(jCheckWebConvertUpdated, props, "WebConvertUpdated");
			setPropsSelected(jCheckWebModifiedOnly, props, "WebModifiedOnly");
			setPropsSelected(jCheckWebModifiedTail, props, "WebModifiedTail");
			setPropsSelected(jCheckWebBeforeChapter, props, "WebBeforeChapter");
			setPropsIntText(jTextWebBeforeChapterCount, props, "WebBeforeChapterCount");
		}
	}
}
