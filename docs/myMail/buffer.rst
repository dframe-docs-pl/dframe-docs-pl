Stworzenie buforu pozwala nam stworzyć kolejkowanie maili dzięki czemu możemy wraz z dframe/database ustawić by był limit wysyłanych maili na godzinę. W tym celu gdy posiadamy już pobranego i zainstalowanego frameworka, tworzymy model o nazwie mail.php

**Model/Mail.php**

.. code-block:: php

 
.. code-block:: php

 namespace Model;
 use Dframe\Config;
 use Dframe\MyMail\MyMail;
  
 class MailModel extends \Model\Model
 {
  
     /**
      * @var array
      */
     protected $buffer;
  
     public function init()
     {
         $this->buffer = array();
     } 
  
     /**
      * @param array $whereArray
      *  return int
      */
     public function mailsCount($whereObject = array())
     {
  
         $query = $this->baseClass->db->prepareQuery('SELECT COUNT(*) AS `count` FROM `mails`');        
         $query->prepareWhere($whereObject);
  
         $row = $this->baseClass->db->pdoQuery($query->getQuery(), $query->getParams())->result();
         return $row['count'];
     }
  
  
     /**
      * @param int    $start
      * @param int    $limit
      * @param array  $whereObject
      * @param string $order
      * @param sting  $sort
      *  return array
      */
     public function mails($start, $limit, $whereObject, $order = 'id', $sort = 'DESC')
     {
  
         $query = $this->baseClass->db->prepareQuery(
             'SELECT mail.*, 
                 `users`.`id`
             FROM  mail 
             LEFT JOIN users ON `mail`.`mail_address` = `users`.`email`'
         );        
         $query->prepareWhere($whereObject);
         $query->prepareOrder($order, $sort);
         $query->prepareLimit($limit, $start);
  
         $results = $this->baseClass->db->pdoQuery($query->getQuery(), $query->getParams())->results();
         return $this->methodResult(true, array('data' => $results));
     }
  
     /**
      * @param array  $address
      * @param string $subject
      * @param string $body
      * @param string $sender
      *  return array
      */
    public function addToBuffer(array $address, $subject, $body, $sender = '', array $attachmentsIds = array())
    {

        $dateUTC = new \DateTime("now", new \DateTimeZone("UTC"));
        
        $mailEntry = array(
            'mail_name' => $address['name'],
            'mail_address' => $address['mail'],
            'mail_subject' => $subject,
            'mail_enqueued' => time(),
            'mail_body' => $body,
            'mail_sender' => $sender,
            'mail_status' => 0,
            'mail_buffer_date' => $dateUTC->format('Y-m-d H:i:s')
        );

        //if($attachment != false){
        //    $mailEntry['mail_attachments_ids'] = $attachmentsIds; 
        //}

        $this->buffer[] = $mailEntry;

        return $this->methodResult(true);
    }
  
     /**
      * return array
      */
    public function execute()
    {
        //Pusty 
        if(count($this->buffer) == 0){
            return $this->methodResult(false, array('response' => 'Buffer is empty'));
        }

        $i = 0;
        try {

            foreach ($this->buffer as $key => $value) {
    
                $buffer = array(
                    'mail_name' => $value['mail_name'],
                    'mail_address' => $value['mail_address'],
                    'mail_subject' => $value['mail_subject'],
                    'mail_enqueued' => $value['mail_enqueued'],
                    'mail_body' => $value['mail_body'],
                    'mail_sender' => $value['mail_sender'],
                    'mail_status' => $value['mail_status'],
                    'mail_buffer_date' => $value['mail_buffer_date']
                );

                $insertResult = $this->baseClass->db->insert('mails', $buffer, true)->getLastInsertId();
                if ($insertResult > 0) {
                    throw new Exception("Filed to add mail", 1);
                }
                
                // Adding attachments do mysql
                // if (isset($buffer['mail_attachments_ids'])){

                //     $attachments = array();
                //     foreach ($buffer['mail_attachments_ids'] as $key2 => $value2) {
                //         $attachments[] = array(
                //             'mail_id' => $insertResult,
                //             'file_id' => $value['mail_attachments_ids']
                //         );
                //     }

                //     $insertAttachmentsResult = $this->baseClass->db->insertBatch('mails_attachments', $attachments)->getLastInsertId();
                //     if(count($insertAttachmentsResult)){
                //         throw new Exception("Filed to add attachment", 1);
                //     }
                // }

                $i++;
            }

        } catch (Exception $e) {
           return $this->methodResult(false, array('response' => $e->getMessage()));
        }
        
        if(!count($i)){
            return $this->methodResult(false, array('response' => 'Unable to add mails to spooler'));
        }
        
        $this->buffer = array();
        return $this->methodResult(true);
    }
  
     /**
      * @param int $amount
      */
    public function sendMails($amount = 20)
    {

        $amount = (int)$amount;
        if($amount <= 0)
            return $this->methodResult(false, 'Incorrect amount');

        $emailsToSend = $this->baseClass->db->pdoQuery('SELECT * 
                                                        FROM `mails` 
                                                        WHERE `mail_status` = ?
                                                        ORDER BY `mail_enqueued` ASC
                                                        LIMIT ?', array('0', $amount))->results();

        $data = array('sent' => 0, 'failed' => 0, 'errors' => array());
        $return = true;

        $mail = new myMail();
        $mail->mailObject->isSMTP();
        $mail->mailObject->SMTPOptions = array(
            'ssl' => array(
                'verify_peer' => false,
                'verify_peer_name' => false,
                'allow_self_signed' => true
            )
        );
        //$mail->SMTPDebug  = 2; // enables SMTP debug information (for testing)
                                 // 1 = errors and messages
                                 // 2 = messages only
        $mail->mailObject->SMTPSecure = false;

        foreach($emailsToSend as $email){

        	$dateUTC = new \DateTime("now", new \DateTimeZone("UTC"));
            try{
                
                //$mailsAttachments = $this->baseClass->db->pdoQuery('SELECT * FROM `mails_attachments` LEFT JOIN files ON mails_attachments.file_id = files.file_id WHERE mail_id = ?', array($email['mail_id']))->results();
                //if (count($mailsAttachments) > 0) {
                    
                    //foreach ($mailsAttachments as $key => $attachment) {
                    //
                    //    $fileStorage = new \Dframe\FileStorage\Storage($this->loadModel('FileStorage/Drivers/DatabaseDriver'));
                    //
                    //    $sourceAdapter = $attachment['file_adapter'].'://'. $attachment['file_path'];
                    //
                    //    if ($fileStorage->manager->has($sourceAdapter)) {
                    //        // Retrieve a read-stream
                    //        $stream = $fileStorage->manager->readStream($sourceAdapter);
                    //
                    //        $contents = stream_get_contents($stream);
                    //        $mail->mailObject->addStringAttachment($contents, end(explode('/', $attachment['file_path'])));
                    //        fclose($stream);
                    //             
                    //    } else {
                    //        throw new \Exception("Brak załacznika", 1);
                    //    }
                    //
                    //    
                    //}

                //}

                $addAddress = array('mail' => $email['mail_address'], 'name' => $email['mail_name']);
                $sendResult =  $mail->send($addAddress, $email['mail_subject'], $email['mail_body']);

                $this->baseClass->db->update('mails', array('mail_sent' => time(), 'mail_status' => '1', 'mail_send_date' => $dateUTC->format('Y-m-d H:i:s')), array('mail_id' => $email['mail_id']));
                $data['sent']++;


            } catch(\Exception $e){
                $data['errors'][] = $e->getMessage();
            }

            if(!isset($sendResult)){
                $data['failed']++;
                $return = false;
                continue;
            }

        }

        //var_dump($data);
        return $this->methodResult($return, $data);
    }
  
     /**
      * return array
      */
     public function clear()
     {
         $this->baseClass->db->truncate('mails');
         return $this->methodResult(true);
     }
  
 }


Cron jest odpalany z poziomu konsoli, służy do cyklicznego sprawdzania czy jest coś do wysłania. Jeśli natrafi na wpis w bazie z mailem to postara się go nam wysłać.

**bin/SmallCron.php**

.. code-block:: php

 set_time_limit(0);
 ini_set('max_execution_time', 0);
 date_default_timezone_set('Europe/Warsaw');
 
 use Dframe\Core;
 
 require_once dirname(__DIR__).'/../vendor/autoload.php';
 require_once dirname(__DIR__).'/../web/config.php';
 require_once dirname(__DIR__).'/../app/Bootstrap.php';
 $bootstrap = new Bootstrap();
   
 class CronSmall extends \Dframe\Controller
 {
     
     public function init()
     {
         $this->dirLog = dirname(__DIR__).'/../web/cache/logs/cronSmall.txt';
    
         if(file_exists($this->dirLog) AND filemtime($this->dirLog)+59 > time()) { 
             echo filemtime($this->dirLog)."\n\r";
             echo time()."\n\r";
             die('Time Limit. Max 59 request on seconds.');
         } 
 
         $this->mailCron();
     }
  
     private function mailCron()
     {
         echo '#Updating mailCron'."\n\r";
         $mailModel = $this->loadModel('mail');
         $mailModel->sendMails();
     }
 
 }
 
 $cron = new CronSmall($bootstrap);
 $cron->init();
 echo 'Ok';

Ostatnim naszym elementem jest kod dodający do bazy. Metoda |addToBuffer| służy do zbierania listy mailingowej a następnie po skończeniu wykonujemy |execute| która dodaje do bazy listę i ją kolejkuje.

.. code-block:: php

 $mailModel = $this->loadModel('Mail');
 $mailModel->addToBuffer(array('name' => 'NameRespondent', 'mail' => 'respondent@Email_respondent'), 'Subjectname', $body);
 $execute = $mailModel->execute();

.. |addToBuffer| cCode:: $mailModel->addToBuffer
.. |execute| cCode:: $mailModel->execute
