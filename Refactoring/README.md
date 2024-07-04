# Рефакторинг проекта по генерации предложений

Ранее проект представлял собой один грубый модуль `SentenceGenerator`, который инкапсулировал в себе всю логику по генерации предложений. Такой модуль было сложно читать и поддерживать. 

Произвел рефакторинг и провел явные границы внутри данного модуля. 

1. Теперь мы концептуально работает с несколькими частями программы:
    - конфиг `Config` отвечает за первоначальную конфигурацию путей к файлам
    - рандомайзер слов `WordRandomizer` отвечает за получение случайного слова
    - словарь для частей речи `WordsParts` отвечает за словари по разным частям речи
    - генерация предложений `Sentence` непосредственно генерирует предложение по заданным правилам
2. Сейчас проект поделен на логические части - его удалось разделить явно, выделяя более высокие абстракции. 

До:
~~~C#
using System;
using System.IO;
using System.Linq;
using System.Text;

namespace GeneratorUtil
{
    public class SentenceGenerator
    {
        private const int BufferSize = 64 * 1024;
        
        private const string RecordSeparator = ". ";

        private static readonly char DirSeparator = Path.DirectorySeparatorChar;

        private readonly string _path =
            $@"{DirSeparator}..{DirSeparator}..{DirSeparator}..{DirSeparator}..{DirSeparator}TxtFiles{DirSeparator}generated.txt";

        private static Random _random;

        private static readonly string DictionaryPathNouns =
            $@"..{DirSeparator}..{DirSeparator}..{DirSeparator}Dictionaries{DirSeparator}nouns.txt";

        private static readonly string DictionaryPathAdverbs =
            $@"..{DirSeparator}..{DirSeparator}..{DirSeparator}Dictionaries{DirSeparator}adverbs.txt";

        private static readonly string DictionaryPathAdjectives =
            $@"..{DirSeparator}..{DirSeparator}..{DirSeparator}Dictionaries{DirSeparator}adjectives.txt";

        private readonly string[] _nouns;
        private readonly string[] _adverbs;
        private readonly string[] _adjectives;

        public SentenceGenerator(string path = null)
        {
            if (!string.IsNullOrEmpty(path))
                _path = path;

            _random = new Random();

            _nouns = File.ReadAllLines(DictionaryPathNouns);
            _adverbs = File.ReadAllLines(DictionaryPathAdverbs);
            _adjectives = File.ReadAllLines(DictionaryPathAdjectives);

            if (!_nouns.Any() || !_adverbs.Any() || !_adjectives.Any())
            {
                throw new ArgumentException($@"Dictionaries wasn't found. 

Check if dictionaries exists:
- Nouns path: {DictionaryPathNouns}
- Adverbs path: {DictionaryPathAdverbs}
- Adjectives path: {DictionaryPathAdjectives}");
            }
        }

        public void Generate(long mb)
        {
            var byteSum = 1024 * 1024 * mb;

            if (byteSum < 0)
                throw new ArgumentException("Provide less size.");

            var random = new Random();
            
            using (var sw = new StreamWriter(_path, false, Encoding.UTF8, BufferSize))
            {
                string cachedSentence = null;

                while (byteSum > 0)
                {
                    var generatedSentence = GenerateSentence();

                    if (cachedSentence != null && byteSum % _random.Next(10, 20) == 0)
                    {
                        generatedSentence = cachedSentence;
                    }

                    var generatedLine = string.Join(RecordSeparator, new string[2]
                    {
                        random.Next(1, 1000).ToString(),
                        generatedSentence
                    });

                    sw.WriteLine(generatedLine);

                    byteSum = byteSum - generatedLine.Length - 1;

                    cachedSentence = generatedSentence;
                }
            }
        }

        private string GenerateSentence()
        {
            return $"{GetRandomWord(_nouns)} is {GetRandomWord(_adverbs)} {GetRandomWord(_adjectives)}";
        }

        private static string GetRandomWord(string[] dictionary)
        {
            return dictionary[_random.Next(1, dictionary.Length - 1)];
        }
    }
}
~~~

После:
~~~C#
using System;
using System.IO;
using System.Linq;
using System.Text;

namespace Generator
{
    public static class Config
    {
        public const int BufferSize = 64 * 1024;

        public const string RecordSeparator = ". ";

        public static string GetPath()
        {
            return $@"{DirSeparator}..{DirSeparator}..{DirSeparator}..{DirSeparator}..{DirSeparator}TxtFiles{DirSeparator}generated.txt";
        }

        private static readonly char _dirSeparator = Path.DirectorySeparatorChar;
    }

    public class WordRandomizer
    {
        public static string Random(string[] dictionary)
        {
            return dictionary[_random.Next(1, dictionary.Length - 1)];
        }
    }

    public class WordsParts
    {
        private readonly string[] _nouns;
        private readonly string[] _adverbs;
        private readonly string[] _adjectives;

        public WordsParts(string[] nouns, string[] adverbs, string[] adjectives)
        {
            _nouns = File.ReadAllLines(nouns);
            _adverbs = File.ReadAllLines(adverbs);
            _adjectives = File.ReadAllLines(adjectives);

        }

        public string[] Nouns => _nouns;
        public string[] Adverbs => _adverbs;
        public string[] Adjectives => _adjectives;

        public bool PartsValid()
        {
            return !_nouns.Any() || !_adverbs.Any() || !_adjectives.Any();
        }
    }

    public class Sentence
    {
        private static Random _random;
        private WordsParts _wordsParts;

        public Sentence(WordsParts wordsParts, string path = null)
        {
            if (!string.IsNullOrEmpty(path))
                _path = path;

            _random = new Random();

            _wordsParts = wordsParts;
        }

        public void Generate(long mb)
        {
            var byteSum = 1024 * 1024 * mb;

            if (byteSum < 0)
                throw new ArgumentException("Provide less size.");

            var random = new Random();

            using (var sw = new StreamWriter(_path, false, Encoding.UTF8, BufferSize))
            {
                string cachedSentence = null;

                while (byteSum > 0)
                {
                    var generatedSentence = Create();

                    if (cachedSentence != null && byteSum % _random.Next(10, 20) == 0)
                    {
                        generatedSentence = cachedSentence;
                    }

                    var generatedLine = string.Join(ConfigRecordSeparator, new string[2]
                    {
                        random.Next(1, 1000).ToString(),
                        generatedSentence
                    });

                    sw.WriteLine(generatedLine);

                    byteSum = byteSum - generatedLine.Length - 1;

                    cachedSentence = generatedSentence;
                }
            }
        }

        private string Create()
        {
            if (!_wordsParts.PartsValid())
            {
                return "";
            }

            return $"{WordRandomizer.Random(_wordsParts.Nouns)} is {WordRandomizer.Random(_wordsParts.Adverbs)} {WordRandomizer.Random(_wordsParts.Adjectives)}";
        }
    }
}
~~~