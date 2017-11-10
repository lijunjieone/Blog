help="$0  -a -b"
if [ $# == 0 ]; then
    echo $help
    exit
fi

while getopts "a:b" arg #选项后面的冒号表示该选项需要参数
do
        case $arg in
             a)
                channelid=$OPTARG
                ;;
             b)
                proguard=1
                ;;
           
             ?)  #
                echo $help
                exit 1
                ;;
        esac
done
