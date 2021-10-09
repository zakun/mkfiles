## PHP Spreadsheet 导出 excel 服务封装
### 1. 文件代码如下:

	<?php

	use PhpOffice\PhpSpreadsheet\Spreadsheet;
	use PhpOffice\PhpSpreadsheet\Writer\Xlsx;

	class ExcelExportService
	{
	    /**
	     * @var Spreadsheet
	     */
	    public $excel;
	    public $fileName;
	    public $rootPath;
	    public $storeDir = '/Files/Export/Tmp/';
	    public $rowNum;
	    public $header = array();
	    public $body = array();

	    public $fields = array();
	    public $values = array();

	    public function __construct($fileName)
	    {
		$this->excel = new Spreadsheet();

		$arrFile = explode('.', $fileName);
		$ext = $arrFile[1];
		$baseName = $arrFile[0];

		$this->fileName = $baseName . '_' . time() . random_int(1000, 9999) . '.' . $ext;
		$this->rowNum = 1;
	    }

	    /**
	     * 设置根目录
	     * @param null $path
	     */
	    public function setRootPath($path = null)
	    {
		if (!empty($path)) {
		    $this->rootPath = rtrim($path, '/');
		} else {
		    $this->rootPath = rtrim(fcommon::FilePath(), '/');
		}
	    }

	    /**
	     * 设置文件存放目, 相对于根目录
	     * @param $pathName
	     */
	    public function setStorePath($pathName)
	    {
		if (!empty($pathName)) {
		    $this->storeDir = rtrim($pathName, '/') . '/';
		} else {
		    $this->storeDir = $this->storeDir . date('Y') . '/' . date('m') . '/';
		}
	    }

	    /**
	     * 设置导出excel 头部
	     * @param $header
	     */
	    public function setHeader($header)
	    {
		$this->header = $header;
	    }

	    /**
	     * 设置导出excel 数据
	     * @param $body
	     */
	    public function setBody($body)
	    {
		$this->body = $body;
	    }

	    /**
	     * excel 写行数据
	     * @param $row
	     * @throws \PhpOffice\PhpSpreadsheet\Exception
	     */
	    public function writeRow($row)
	    {
		$sheet = $this->excel->getActiveSheet();
		$column = 0;
		foreach ($row as $k => $value) {
		    $sheet->setCellValueByColumnAndRow(++$column, $this->rowNum, $value);
		}

		$this->rowNum++;
	    }

	    /**
	     * excel 写表头
	     * @throws \PhpOffice\PhpSpreadsheet\Exception
	     */
	    public function writeHeader()
	    {
		$this->writeRow($this->header);
	    }

	    /**
	     * excel 写表数据
	     * @throws \PhpOffice\PhpSpreadsheet\Exception
	     */
	    public function writeBody()
	    {
		foreach ($this->body as $k => $row) {

		    $filterRow = [];
		    foreach($this->fields as $field)
		    {
			$value = $row[$field];
			if(isset($this->values[$field]) && is_callable($this->values[$field]))
			{
			    $value = call_user_func($this->values[$field], $value, $row);
			}
			$filterRow[] = $value;
		    }

		    $this->writeRow($filterRow);
		}
	    }

	    /**
	     * 解析表头, 和 导出字段值得自定义设置
	     * columns 格式 :
	     * 匿名函数的参数: $value, 导出字段对应的原始值, $row, 导出原始字段所在的行数组
	     * $columns = [
	     *      'id'=>'序号',
	     *      'name'=>[
	     *          'label'=>'姓名',
	     *          'value'=>function($value, $row){
	     *              return $value
	     *          }
	     *      ]
	     * ]
	     * @param $columns array
	     */
	    public function parseColumns($columns)
	    {
		$header = [];
		$fields = [];
		$values = [];
		foreach ($columns as $k => $item) {
		    $fields[] = $k;

		    if (is_string($item)) {
			$header[] = $item;

		    } elseif (is_array($item)) {
			$header[] = $item['label'];
			if(isset($item['value']) && is_callable($item['value']))
			{
			    $values[$k] = $item['value'];
			}
		    }
		}

		$this->header = $header;
		$this->fields = $fields;
		$this->values = $values;
	    }

	    /**
	     * excel 设置列宽
	     * @param $widths
	     * @throws \PhpOffice\PhpSpreadsheet\Exception
	     */
	    public function setWidths($widths)
	    {
		if(count($widths) == count($this->header))
		{
		    for($i=1; $i<=count($this->header); $i++)
		    {
			$this->excel->getActiveSheet()->getColumnDimensionByColumn($i)->setWidth($widths[$i-1]);
		    }
		}
	    }

	    /**
	     * 设置导出文件的保存目录
	     * @param $path string 相对于根目录的路径
	     * @return string
	     */
	    public function setFilePath($path)
	    {
		$this->setRootPath();
		$this->setStorePath($path);
		$absPath = $this->rootPath . $this->storeDir . $this->fileName;

		$arrPath = pathinfo($absPath);
		if (!is_dir($arrPath['dirname'])) {
		    mkdir($arrPath['dirname'], 0777, true);
		}

		return $absPath;
	    }

	    /**
	     * 导出excel文件
	     * @param null $path string 保存文件的目录, 相对于根目录
	     * @throws \PhpOffice\PhpSpreadsheet\Exception
	     * @throws \PhpOffice\PhpSpreadsheet\Writer\Exception
	     */
	    public function save($path = null)
	    {
		if (empty($this->body)) {
		    throw new Exception('没有待导出的数据', 501);
		}

		//设置文件保存目录
		$absPath = $this->setFilePath($path);

		//write header
		$this->writeHeader();

		//write body
		$this->writeBody();

		$write = new Xlsx($this->excel);
		$write->save(fcommon::linuxStringUtf($absPath));

		$this->excel->disconnectWorksheets();
		unset($this->excel);
	    }
	    //自定义文件下载路径
	    public function getFullFileUrl()
	    {
		$baseUrl = fcommon::FileUrl();

		$fileUrl = $baseUrl . $this->storeDir . $this->fileName;

		return $fileUrl;
	    }
	}

### 2. 使用方式如下:

	...

	$fileName = '教师数据' . date('YmdHis') . '.xlsx';
	$width = ['20', '20', '20', '10', '10', '10', '15', '15', '15', '10', '15', '15', '20', '20', '20', '20', '20', '10', '10', '10', '10', '10', '10', '10', '10'];

	$excel = new ExcelExportService($fileName);

	$excel->parseColumns($columns);
	$excel->setWidths($width);
	$excel->setBody($exportData);
	$excel->save();

	$fullUrl = $excel->getFullFileUrl();

	...

