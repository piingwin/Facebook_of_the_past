# Facebook_of_the_past

import sys
reload(sys)
sys.setdefaultencoding('utf-8')

import re
from lxml import etree
from os import makedirs

global folder
folder = u'done'

global lfolder
lfolder = u'%s/letters' % folder

global table2
table2 = []

class Letters():
    def __init__(self):
      self.list_of_letters = []
      
      
    def add_letter(self, letter):
      self.list_of_letters.append(letter)

    def import_list_of_letters_to_file(self, filename):
       h = open(filename, 'w+')
       for p in self.list_of_letters:
         #  id текста#автор письма#адресат письма#дата#место#рус/нерус#автор CC#сайт/издание#том#номер письма#стр#линк на примечание
         #  case_number#имя (ФИО)#id текста#роль (автор, адресат, упоминат)
         h.write("%d#%s#%s#%s#%s#%s#%s#%s#%s#%d#%s#%s\n" % (p.letter_id, 
                                                            p.name, 
                                                            p.adressat, 
                                                            p.date, 
                                                            p.place, 
                                                            p.russ, 
                                                            p.author_cc, 
                                                            p.site, 
                                                            p.tom, 
                                                            p.letter_number, 
                                                            p.page, 
                                                            p.link
                                                           ))
       h.close()

class Letter():
    id = 1
    
    def __init__(self):
        self.letter_id = self.letter_number = Letter.id
        Letter.id = Letter.id + 1
        
        self.name = 'Ф.И.Тютчев'
        self.site = 'http://feb-web.ru'
        self.tom = u'Полное собрание сочинений и писем: В 6 т. Том четвертый: Письма, 1820—1849'
        self.author_cc = 'Редколлегия: Н. Н. Скатов (гл. ред.), Л. В. Гладкова, Л. Д. Громова-Опульская, В. М. Гуминский, В. Н. Касаткина, В. Н. Кузин, Л. Н. Кузина, Ф. Ф. Кузнецов, Б. Н. Тарасов.'
        self.adressat = self.date = self.place = self.russ = self.page = self.link = ''

class TableTwoData():
    case_number = 1
    
    def __init__(self, name, text_id, role):
        self.case_number = TableTwoData.case_number
        TableTwoData.case_number = TableTwoData.case_number + 1
        self.name = name
        self.text_id = text_id
        self.role = role
    
def get_links():
  links = []
  with open('tree.txt', 'r') as f:
    for line in f:
      link_in_block = re.findall('.+<a href="(.+?)".+', line)
      link = 'http://feb-web.ru'+link_in_block[0][:-1]+'2'
      links.append(link)
  return links   

def run():
  try:
      makedirs(u'./%s' % lfolder)
  except:
      pass

  letters = Letters()
  links = get_links()
  #bad_links = []
  
  for link in links:
    i = 1
    while True:
      print link
      try:
        root = etree.parse(link, etree.HTMLParser())
      except:
        print 'coudnot load %s' % link
        print 'trying: %d' % i
        i +=1        
        #bad_links.append(link)
      else:
        letters.add_letter(process(root, link))
        break
      
  return letters

def dative_nominative(name):
    replace = []
    replace.append({'search': u'кому', 'replace': u'кий'})
    replace.append({'search': u'у', 'replace': u''})
    replace.append({'search': u'ской', 'replace': u'ая'})
    replace.append({'search': u'ой', 'replace': u'а'})
    replace.append({'search': u'ым', 'replace': u'ы'})
    replace.append({'search': u'е', 'replace': u''})
    replace.append({'search': u'лю', 'replace': u'ль'})
  
    words = name.split()
    for pattern in replace:
        for i, word in enumerate(words):
            if word.isupper:
                word = word.title()
            if word[-len(pattern['search']):] == pattern['search']:
                words[i] = word[:-len(pattern['search'])] + pattern['replace']
    
    return ' '.join(words)

def language(symbol):
    match = re.search('[A-Za-z]', symbol, re.U)
    if match:
        return u'не русский'
    return u'русский'

#self.text = ''
def process(root, link):
    letter = Letter()
    letter.link = link
  
    try:
      title = root.xpath(".//*[@id='prose']/p[starts-with(@class,'zag10bot')]/a")[0].text
      finded = re.findall('(\d+)\.(.+)', title)
      #letter.letter_id = finded[0][0].strip()
      letter.adressat = dative_nominative(finded[0][1].strip())
    except:
      print 'exception TITLE'
      print link
    
    under_title = root.xpath(".//*[@id='prose']/p[@class='zag10i']")[0].text
    under_title = under_title.encode('utf8')
    sp = under_title.split('г.')
    letter.date = sp[0].strip()
    if len(sp) > 1:
        letter.place = 'г. %s' % sp[1].strip()
    
    all_paragrafs = root.xpath(".//*[@id='prose']/p[@class !='zag10i' and not(starts-with(@class,'zag10bot'))]")
    text = ''
    for p in all_paragrafs:
      try:
        p_iter = p.itertext()
      except:
        print 'exception ITERTEXT'
        print link
      try:
         for i in p_iter:
            text += i.replace('\n', '').replace('#', '')
      except:
         print 'exception!'
         print link

    h = open('%s/%d.txt' % (lfolder, letter.letter_id), 'w+')
    h.write(text)
    h.close()
    
    letter.russ = language(text[0])
    
    pages_list = root.xpath(".//*/span[@class='page']")
    pages = []
    for p in pages_list:
      pages.append(p.text)
    letter.page = '; '.join(pages)
    
    # заполняем 2ю таблицу
    table2.append(TableTwoData(letter.name, letter.letter_id, 'автор'))
    table2.append(TableTwoData(letter.adressat, letter.letter_id, 'адресат'))
    
    return letter
  
if __name__ == '__main__':
    #link = 'http://feb-web.ru/feb/tyutchev/texts/pss06/tu4/tu4-128-.htm?cmd=2'
    #r2 = etree.parse(link, etree.HTMLParser())
    
    letters = run()
    letters.import_list_of_letters_to_file('./%s/table1.csv' % folder)
    
    h = open('./%s/table2.csv' % folder, 'w+')
    for t in table2:
        h.write("%d#%s#%s#%s\n" % (t.case_number, t.name, t.text_id, t.role))
    h.close()
