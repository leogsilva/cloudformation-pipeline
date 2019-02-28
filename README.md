aws cloudformation create-stack --stack-name Project02 --template-body file://templates/cfn-pipeline.yaml --capabilities CAPABILITY_IAM 

cd novorepo
git clone https://git-codecommit.us-east-1.amazonaws.com/v1/repos/Project02 .
cp -rpf ../meurepo/* .
cp -rpf hooks/pre-commit .git/hooks 

git add *
git commit -m "first commit" 
git push origin master
git checkout -b staging master
git add *
git commit -m "first commit at staging" 
git push origin staging   



https://br.atlassian.com/git/tutorials/using-branches/git-merge





