# UpdatecommentsFromjiratoSalesforce

global class JiraIssueCommentUpdate implements Database.batchable<string>,Database.AllowsCallouts,Schedulable{ 
    map<string,string> keyList= new map<string,string>();
    GnrSf__JIRA_Login_Settings__c  setting;
    List<GnrSf__JiraIssueComments__c> jiraCommentListtoInsert = new List<GnrSf__JiraIssueComments__c>();
    List<GnrSf__JiraIssueComments__c> jiraCommentListtoUpdate = new List<GnrSf__JiraIssueComments__c>();
    global JiraIssueCommentUpdate(map<String,string> jirakeyList){
        keyList = jirakeyList; 
    }
    
    global list<string> start(Database.BatchableContext bc){ 
        if (!keyList.isEmpty()) {
            
            list<string> keyList2 = new list<string>(keyList.keyset());
            return keyList2;
        } else {
            return null; 
        }     
    } 
    
    global void execute(Database.BatchableContext bc, list<string> scope){
        try{ 
            setting = [select GnrSf__IsCommentPrivate__c from GnrSf__JIRA_Login_Settings__c where name = 'JIRA' LIMIT 1];
            
            for (string key:scope){
                set<string> commentIdJira = new set<string>();
                for (GnrSf__JiraIssueComments__c com:[SELECT Id,GnrSf__CommentId__c FROM GnrSf__JiraIssueComments__c WHERE GnrSf__JiraIssue__c =: keyList.get(key)]){
                    commentIdJira.add(com.GnrSf__CommentId__c);
                }
            Http http = new Http();
            HttpRequest request = new HttpRequest();
            HttpResponse response = new HttpResponse();
            request = GnrSf.CreateJIRAWrapper.CreateRequest('/rest/api/2/issue/'+key+'/comment','GET', 'application/json');
          //  system.debug('request '+request);
            response = http.send(request);
           // system.debug('response '+response);
            Map < String, Object > results = (Map < String, Object > ) JSON.deserializeUntyped(response.getBody());
            list<object> commentList = (list<object>)results.get('comments');
            if (!commentList.isEmpty()){
                for (object obj:commentList){
                    map<string,object> objMap = (map<string,object>)obj;
                    
                    GnrSf__JiraIssueComments__c comment = new GnrSf__JiraIssueComments__c();
                    comment.GnrSf__CommentBody__c = string.valueOf(objMap.get('body')).stripHTMLTags();
                    comment.GnrSf__CommentId__c = string.valueOf(objMap.get('id')); 
                    if (setting.GnrSf__IsCommentPrivate__c){
                        comment.GnrSf__IsPublished__c = false;  
                    } else{
                        comment.GnrSf__IsPublished__c = true;
                    }
                    comment.GnrSf__JiraIssue__c = keyList.get(key); 
                    comment.GnrSf__UpdateAuthor__c = string.valueOf(((map<string,object>)objMap.get('updateAuthor')).get('key')); 
                    comment.GnrSf__Author__c = string.valueOf(((map<string,object>)objMap.get('author')).get('key'));  
                    comment.GnrSf__Comment_Added__c = true; 
                    comment.GnrSf__Comment_Source__c = 'Jira'; 

                    //  system.debug('comment '+comment);
                   if(!commentIdJira.contains(string.valueOf(objMap.get('id')))){
                    jiraCommentListtoInsert.add(comment);
                   } else {
                      jiraCommentListtoUpdate.add(comment) ;
                   }
                    }
            }
        }
            if(!jiraCommentListtoInsert.isEmpty()){
                //system.debug('jiraCommentList '+jiraCommentList);
                Database.SaveResult[] saveResultList = Database.insert(jiraCommentListtoInsert, true);
                //system.debug('saveResultList '+saveResultList);
            }
            if(!jiraCommentListtoUpdate.isEmpty()){
                Database.SaveResult[] saveResultList = Database.update(jiraCommentListtoInsert, true);
            }
           }catch(Exception ex){
              // system.debug('Exception '+ex.getMessage());
           } 
    } 
    
    global void finish(Database.BatchableContext bc){  
        list<string> listId =new  list<string>();
        for (CronJobDetail CronJob:[SELECT Id FROM CronJobDetail WHERE Name like '%Jira Comment Update%']){
            listId.add(CronJob.id);
        }   
        if (!listId.isEmpty()){
            for(CronTrigger cronTrig:[SELECT Id from CronTrigger WHERE CronJobDetailId = :listId]){
               System.abortJob(cronTrig.Id);  
            }
        }     
    } 
    
    global void execute(SchedulableContext sc) {
        JiraIssueCommentUpdate JiraCommentUpdate = new JiraIssueCommentUpdate(keyList);
        Database.executeBatch(JiraCommentUpdate,1);
    }
}
