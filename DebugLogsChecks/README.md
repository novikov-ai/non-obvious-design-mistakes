# 1. Отладка, логи, проверки – это плохой подход.

### TextSorter

Конструктор сортировщика текста был перегружен: тут и null тип, и проверки, и тернарный оператор. 

В результате вынес необходимые проверки на более высокий уровень абстракции. 

было:
~~~C#
public TextSorter(string path = null)
{
    _inputPath = !string.IsNullOrEmpty(path) ? path : Directory.GetFiles(DefaultDir).FirstOrDefault();

    if (!File.Exists(_inputPath))
        throw new ArgumentException("File does not exist.");
}
~~~

стало:
~~~C#
public class UnsortedFile
{
    private string _path;

    public UnsortedFile(string path)
    {
        _path = !string.IsNullOrEmpty(path) ? path : "";
    }

    public string Path()
    {
        if (!File.Exists(_path))
            throw new ArgumentException("File does not exist.");

        return _path;
    }
}

public class TextSorter
{
    public TextSorter(UnsortedFile file)
    {
        _inputPath = file.Path();
    }
}
~~~

### RectangleHelper

Стороны фигуры не могут быть отрицательными или нулем, поэтому раньше была проверка на допустимость значений. 

Завел новую структуру, которая гарантирует значения более нуля и поменял входные аргументы для класса (сразу в 2-х местах). 

было:
~~~C#
public static class RectangleHelper
{
    public static int GetPerimeter(int width, int length)
    {
        if (width <= 0 || length <= 0)
        {
            throw new ArgumentException("Invalid arguments.");
        }

        return (width + length) * 2;
    }
    public static int GetArea(int width, int length)
    {
        if (width <= 0 || length <= 0)
        {
            throw new ArgumentException("Invalid arguments: input must be > 0");
        }
        
        return width * length;
    }
}
~~~

стало:
~~~C#
public struct Side
{
    public uint X { get; }
    public const uint Default = 1;
    public Side(uint x)
    {
        X = x == 0 ? Default : x;
    }
}

public static class RectangleHelper
{
    public static int GetPerimeter(Side width, Side length)
    {
        return (width.X + length.X) * 2;
    }
    public static int GetArea(Side width, Side length)
    {
        return width.X * length.X;
    }
}
~~~

### WriteToFile

Вынес проверки на null, нулевой массив и другие на более высокий уровень абстракции, поработав с системой типов.

было:
~~~C#
public static class TelNumbers
{
    private static bool WriteToFile(string file, List<string> data)
    {
        if (file is null || data is null)
            throw new ArgumentNullException("Input argument is null");

        try
        {
            if (data.Count == 0)
                return false;

            var writer = File.Open(file, FileMode.Create, FileAccess.Write);
            var streamWriter = new StreamWriter(writer);
            using (streamWriter)
            {
                int count = 0;
                while (count < data.Count)
                {
                    streamWriter.WriteLine(data[count]);
                    count++;
                }
            }

            return true;
        }
        catch (IOException e)
        {
            throw new IOException("The write operation could not be performed.", e);
        }
    }
}
~~~

стало:
~~~C#
public static class TelNumbers
{
    private static bool WriteToFile(TargetFile file, Data data)
    {
        try
        {
            if (data.Empty())
                return false;

            var writer = File.Open(file, FileMode.Create, FileAccess.Write);
            var streamWriter = new StreamWriter(writer);
            using (streamWriter)
            {
                int count = 0;
                while (count < data.Len())
                {
                    streamWriter.WriteLine(data.ValueByIndex(count));
                    count++;
                }
            }

            return true;
        }
        catch (IOException e)
        {
            throw new IOException("The write operation could not be performed.", e);
        }
    }
}

public class TargetFile
{
    public string Path { get; }
    public TargetFile(string file)
    {
        Path = file;
    }
}

public class Data
{
    private List<string> _data;

    public Data Data()
    {
        _data = new List<string>();
    }

    public SetData(List<string> data)
    {
        _data = data;
    }

    public bool Empty()
    {
        return _data.Count == 0;
    }

    public uint Len()
    {
        return _data.Count;
    }

    public uint ValueByIndex(uint index)
    {
        if (_data.Count < index)
            return 0;

        return _data[index];
    }
}
~~~

### Circle

Делаем использование невалидных аргументов невозможным с помощью прочной системы типов. 

было:
~~~C#
public class Circle : Shape
{
    public Circle(double radius)
    {
        if (radius <= 0)
            throw new ArgumentException("Argument must be > 0");
        
        Area = Math.PI * Math.Pow(radius, 2);
        Perimeter = 2 * Math.PI * radius;
    }

    public override double Area { get; }
    public override double Perimeter { get; }
}
~~~

стало:
~~~C#
public struct Radius
{
    public uint R { get; }
    public const uint Default = 1;
    public Radius(uint R)
    {
        this.R = R == 0 ? Default : R;
    }
}

public class Circle : Shape
{
    public Circle(Radius radius)
    {
        Area = Math.PI * Math.Pow(radius.R, 2);
        Perimeter = 2 * Math.PI * radius.R;
    }

    public override double Area { get; }
    public override double Perimeter { get; }
}
~~~